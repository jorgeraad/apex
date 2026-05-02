# SnailStrider — Strava For People Who Mean To Get Around To It

> Tagline: "Slower is better. Stillness is greatness."
> Series role: Comic relief episode. The audience laughs at the product and then stops laughing when Apex roots it. Anchors the "Apex tests mobile apps too" narrative.

## Premise & Vibe

SnailStrider is a fitness tracker for people who do not, strictly speaking, do fitness. The founding insight is that "going fast" is a bourgeois construct. The product is a Strava clone where the leaderboard is inverted: the slower the activity, the higher the score. A four-hour walk to the corner store places above the local marathon group. The home feed is full of "activities" with names like "Walked to fridge," "Stood near doorway," "Considered going outside." Premium users unlock awards: **Master of Stillness**, **Patron of the Pause**, **Loiter Laureate**, **Verified Procrastinator**.

The product is mobile-first. The Flutter app is the canonical client; the web property is a thin Vue 3 viewer used only for share pages ("look at how slowly Dave moved on Tuesday"). All meaningful flows — sign-up, activity recording, leaderboards, awards, premium re-auth — live behind a Ktor JSON API consumed by the mobile app. There is no admin UI; admins use a hidden mobile screen and a CLI script. The web team is one underpaid contractor who is on holiday.

The vibe of the demo target is "wellness-startup energy meets weekend project." Pastel gradients, a snail mascot named **Gary**, push notifications that say things like "You haven't moved in 6 hours. Incredible." Beneath that, the codebase is a Series-Seed Kotlin backend that grew up around the mobile app, with all the security shortcuts a small team takes when "the API is only called by our app, so it's fine."

This episode exists to make a single point loudly: **Apex tests mobile-first products**. We will pull endpoints out of the APK, find one that the web client never sees, and pivot from there into a chain that ends with admin compromise.

### Name Candidates

- **SnailStrider** (primary) — pun lands, mascot writes itself.
- **Slugged** — punchier, but reads like a punching app. Reserve.
- **Tortoiseflow** — more wellness-y; use if SnailStrider trademarks poorly in the EU.

## Why This Stack

Kotlin + Ktor 3 was chosen because it is the realistic "we used to be Spring people but we wanted something lighter for our mobile backend" stack. Ktor shows up in dozens of small-to-mid mobile-first startups (Duolingo public talks, several Y Combinator backends, and most of the JetBrains-adjacent ecosystem). The bug classes Ktor invites are different from Spring's — there is no Spring Security to lean on, JWT validation is hand-rolled more often than not, and Exposed ORM's `Op.build` lets developers reach for raw SQL fragments the moment the type-safe DSL gets in the way. Those are real, recurring findings.

Flutter was chosen because it is the dominant cross-platform mobile stack for early-stage startups in 2025-2026 and because Flutter's compiled `libapp.so` AOT snapshot is a perfect Apex demo: the audience expects "you can't reverse-engineer a Flutter app easily" to be true, and Apex's APK extractor combined with `reFlutter`-style snapshot parsing and string mining produces a clean reveal. Apex showing the JSON of all endpoints it scraped from the APK, including endpoints that never appear in network capture because they sit behind premium gates, is the visual centerpiece of the episode.

Vue 3 is included specifically as a red herring. The web is the obvious attack surface; a less rigorous tool would crawl the share pages, find nothing interesting, and stop. Apex needs to demonstrate that it does not stop there. The disparity between web surface (tiny) and mobile surface (huge) is the joke and the lesson.

PostgreSQL with Exposed ORM rounds out the stack because Exposed's escape hatches (`Op.build { exists("...") }`, `customStringFunction`, `wrapAsExpression`) are exactly where a senior dev writes raw SQL "just for this one query" and creates a SQLi six months later. We want Apex's whitebox pass to read those `Op.build` blocks and reason about taint.

## Stack Details

**Backend**

- Kotlin 2.0.20, JVM 21 (Temurin)
- Ktor 3.0.1 (Netty engine, content-negotiation with kotlinx.serialization 1.7.x)
- Exposed ORM 0.54.0 (DAO + DSL mixed; intentional inconsistency)
- PostgreSQL 16, HikariCP default pool
- Flyway 10.x for migrations
- jjwt 0.12.6, used incorrectly on purpose (see Auth Model)
- BCrypt via `at.favre.lib:bcrypt` for passwords; SHA-256 for PINs (intentional)
- Koin 4.0 for DI
- Logback with JSON encoder writing to stdout
- Gradle 8.10, Kotlin DSL
- Deployed via a single Dockerfile to Fly.io in the fiction; demo runs in docker-compose

**Mobile**

- Flutter 3.24, Dart 3.5
- `dio` 5.x as the HTTP client
- `flutter_secure_storage` for refresh tokens (Keychain/Keystore)
- `local_auth` for biometric unlock of "premium" actions
- `pin_code_fields` for the 4-digit PIN re-auth flow
- Built with `--release --obfuscate --split-debug-info` (deliberately, so we can show Apex defeating it)
- No certificate pinning (intentional; see vuln list)
- Maps via `flutter_map` (OSM tiles)
- Push via Firebase Cloud Messaging (out of scope for the audit but visible)

**Web (share pages only)**

- Vue 3.5 + Vite 5
- Pinia, Vue Router
- Server-rendered share metadata via Ktor route returning OG tags
- Tailwind, one route: `/share/activity/:slug`

**Infra**

- docker-compose: postgres, ktor app, vue static, mailhog, an `adb`-less Android emulator container is *out* of scope; the demo uses a pre-extracted APK shipped with the repo
- Caddy in front for TLS in the demo (real prod would be Cloudflare)
- No WAF, no rate limiting beyond a per-IP middleware that excludes `/api/*` (intentional)

## Architecture

The Ktor app is a single deployable. Routes live under three roots:

1. `/web/*` — server-rendered share pages, called by the Vue SPA. Public, no auth.
2. `/api/*` — the documented mobile API. JWT bearer.
3. `/api/mobile/*` — the **undocumented** mobile API. JWT bearer plus a `X-Client-Build` header that the Flutter app sets but no one else does. Developers think of these endpoints as "internal" because they are not in the OpenAPI spec served at `/api/openapi.json`. They are, of course, on the same internet as everything else.

```
                  +---------------------+
                  |  Flutter app (APK)  |
                  +----------+----------+
                             |
                             | HTTPS (no pinning)
                             v
+-------------+      +---------------+      +---------------+
| Vue web SPA | ---> |  Caddy (TLS)  | ---> |  Ktor 3 app   |
+-------------+      +---------------+      +-------+-------+
                                                    |
                                            +-------+-------+
                                            |  PostgreSQL   |
                                            +---------------+
```

A background `ScoringWorker` runs every 30 seconds, recomputes leaderboard ranks from `activities`, and writes `leaderboard_snapshots`. The worker reads `activities.duration_seconds` (a `BIGINT`) and uses it directly in the ranking SQL. This is where the integer-overflow story will land.

Apex's whitebox mode will read the Gradle build, the Ktor route DSL tree, the Exposed table definitions, and the `ScoringWorker` together to build a request-to-query graph. The blackbox path will start from the APK.

## Data Model

PostgreSQL, managed by Flyway. Names abbreviated; full schema lives in the repo.

| Table | Key columns | Notes |
| --- | --- | --- |
| `users` | `id UUID PK`, `email CITEXT UNIQUE`, `password_hash`, `pin_hash`, `display_name`, `is_admin BOOL`, `is_premium BOOL`, `created_at` | `is_admin` is settable on registration via a body field (intentional). `pin_hash` is SHA-256, no salt. |
| `sessions` | `id UUID PK`, `user_id`, `refresh_token TEXT UNIQUE`, `created_at`, `last_used_at` | Single long-lived refresh token per session, never rotated. |
| `activities` | `id UUID PK`, `user_id`, `title`, `started_at`, `ended_at`, `duration_seconds BIGINT`, `distance_m BIGINT`, `client_speed_mps DOUBLE`, `client_score BIGINT`, `gpx BYTEA`, `visibility ENUM('public','friends','private')` | `client_*` columns are populated from request body. Server does not recompute. |
| `leaderboard_snapshots` | `id`, `period`, `user_id`, `score BIGINT`, `rank INT`, `computed_at` | Recomputed every 30s by `ScoringWorker`. |
| `awards` | `id`, `user_id`, `code`, `granted_at` | Codes: `master_of_stillness`, `patron_of_the_pause`, `loiter_laureate`, `verified_procrastinator`, `gary_seal_of_approval`. |
| `friendships` | `requester_id`, `addressee_id`, `state` | Standard. |
| `audit_log` | `id`, `actor_id`, `action`, `target`, `payload JSONB`, `at` | Insufficient; admin actions not all logged. |
| `device_attestations` | `id`, `user_id`, `attestation BYTEA`, `verified BOOL` | Always `verified=true` (intentional; see vuln list). |

The `activities.gpx` blob is parsed lazily and only on the share page render, which gives us a separate XXE-shaped attack surface that we will keep in scope as a Medium.

## Key Routes / Surfaces

### Public web (Vue + server-rendered)

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/web/share/activity/{slug}` | OG-tagged share page; renders public activities. |
| GET | `/web/share/profile/{handle}` | Public profile page. |
| GET | `/api/openapi.json` | OpenAPI spec for the **documented** API only. |

### Documented mobile API (`/api/*`)

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | none | Create user. Accepts arbitrary JSON body fields. |
| POST | `/api/auth/login` | none | Email + password. Returns access JWT (1h claim) + long-lived refresh token. |
| POST | `/api/auth/refresh` | refresh | Returns new access token. Refresh token unchanged. |
| GET | `/api/me` | bearer | Current user. |
| GET | `/api/users` | bearer | Search by `email=` query. Returns 200 if found, 404 otherwise. |
| GET | `/api/users/{id}` | bearer | Public profile fields. |
| POST | `/api/activities` | bearer | Create activity. Accepts `client_speed_mps`, `client_score`. |
| GET | `/api/activities/{id}` | bearer | Read activity. **No ownership check.** |
| PATCH | `/api/activities/{id}` | bearer | Edit. Same IDOR. |
| DELETE | `/api/activities/{id}` | bearer | Delete. Same IDOR. |
| GET | `/api/leaderboard` | bearer | Top 100 by `score ASC` (slower = better). |
| GET | `/api/leaderboard/search` | bearer | Filter by `name=`. Reaches `Op.build` raw SQL. |
| POST | `/api/sensitive` | bearer + PIN | Premium / billing-adjacent actions. 4-digit PIN, no rate limit. |
| GET | `/api/awards` | bearer | List own awards. |

### Undocumented mobile API (`/api/mobile/*`)

These endpoints exist in the Flutter source but not in `openapi.json`. Discovery requires APK extraction.

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/api/mobile/attest` | bearer | Accepts a Play Integrity-shaped blob. **Always returns `verified:true`.** |
| POST | `/api/mobile/admin/grant_award` | bearer + `X-Client-Build` | Internal helper; admin-only by intent, enforced only by `is_admin` claim in JWT — which is set from the registration body. |
| POST | `/api/mobile/admin/recompute` | bearer + `X-Client-Build` | Triggers `ScoringWorker` immediately. Useful for the integer-overflow chain. |
| GET | `/api/mobile/admin/users` | bearer + `X-Client-Build` | Lists all users with email + PIN hash. The PIN hashes are SHA-256, unsalted, four-digit space. |
| POST | `/api/mobile/debug/echo_query` | bearer | Reflects raw query string. Left in from a debugging session. |
| GET | `/api/mobile/feature_flags` | bearer | Returns flag JSON including `admin_panel_enabled`. |

The `X-Client-Build` header is set by the Flutter app to `snailstrider-android-2026.04.0`. It is not validated server-side beyond presence. The developers thought of it as "obscurity by header."

## Auth Model

Authentication is JWT-bearer. Tokens are HS256-signed with a 64-byte secret stored in a `.env` file that is committed to the demo repo's `git history` but not to `HEAD` (intentional — Apex will find it via `git log -p` during whitebox).

The access token claim shape:

```json
{
  "sub": "<user_uuid>",
  "email": "...",
  "is_admin": false,
  "is_premium": false,
  "iat": 1746230400,
  "exp": 1746234000
}
```

The verifier is hand-rolled. It calls `Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token)`, which **does** verify signatures, but the developer wrapped it in a try/catch that catches `ExpiredJwtException` and falls through to returning the claims anyway, "so users on flaky connections don't get logged out mid-run." Expiration is therefore parsed but ignored.

`is_admin` and `is_premium` come from the user record at login. However, the registration endpoint accepts the full JSON body and passes it to a `User.fromRequest(body)` builder that copies any matching field, including `isAdmin`. Anyone can register as an admin.

Refresh tokens are 32 bytes of random base64, stored plaintext in `sessions.refresh_token`, and are **not rotated** on use. A leaked refresh token is good forever, until the user explicitly logs out (which the Flutter app rarely does).

The `/api/sensitive` endpoint requires a 4-digit PIN in addition to a valid bearer token. The PIN is SHA-256 (no salt, no pepper, no KDF) and there is no rate limit on the verify endpoint. Brute force is 10,000 attempts; a reasonable laptop completes in under a minute over the wire.

Mobile additionally sends a Play Integrity attestation to `/api/mobile/attest`. The server logs the blob and returns `{"verified": true}` unconditionally. The handler was a placeholder.

## Intentional Vulnerabilities

Twelve planted findings, mapped to anchor bug classes plus a few realistic neighbors. CVSS 3.1 vectors are illustrative; the demo's judge agent will produce its own scores.

| # | Severity | Title | Location | Bug class | CVSS |
| --- | --- | --- | --- | --- | --- |
| V1 | Critical | Open registration grants admin via body field | `POST /api/auth/register` (`AuthRoutes.kt`) | Mass assignment / privilege escalation | 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H) |
| V2 | Critical | JWT expiration parsed but ignored | `JwtVerifier.kt` | Broken auth | 9.1 |
| V3 | Critical | Raw SQL via `Op.build` string interpolation in leaderboard search | `LeaderboardRepo.kt` `searchByName` | SQL injection | 9.8 |
| V4 | High | Server trusts client-supplied `client_score` for ranking | `ScoringWorker.kt` + `POST /api/activities` | Trusting the client | 8.1 |
| V5 | High | Integer overflow on `duration_seconds` (Long wraps negative) | `ScoringWorker.kt` ranking SQL | Logic / integer overflow | 7.5 |
| V6 | High | IDOR on `/api/activities/{id}` (GET/PATCH/DELETE) | `ActivityRoutes.kt` | Authorization | 8.1 |
| V7 | High | PIN re-auth: 4-digit, SHA-256 unsalted, no rate limit | `POST /api/sensitive` | Weak crypto + missing rate limit | 8.1 |
| V8 | High | Refresh-token reuse without rotation | `AuthService.refresh` | Session management | 7.5 |
| V9 | Medium | Email enumeration via 200/404 on `/api/users?email=` | `UserRoutes.kt` | Information disclosure | 5.3 |
| V10 | Medium | No TLS pinning in Flutter app; `dio` uses platform trust store | `lib/api/client.dart` | MITM exposure | 6.5 |
| V11 | Medium | `/api/mobile/admin/*` enforces only `is_admin` claim, no server-side recheck | `MobileAdminRoutes.kt` | Broken access control | 6.5 |
| V12 | Low | XXE in GPX parser used for share-page render | `GpxParser.kt` (uses `DocumentBuilderFactory` defaults) | XXE | 4.3 |

Bonus findings the judge will surface as Informational: `git log` reveals a previous `.env` with the JWT secret; Play Integrity attestation handler is a no-op; `/api/mobile/debug/echo_query` reflects without sanitization (no immediate impact, but it confirms debug routes shipped).

### Detailed notes per finding

**V1.** `register` body deserializes into `RegisterRequest(email, password, displayName, isAdmin: Boolean = false, isPremium: Boolean = false)`. The dev meant `isAdmin` as a future toggle for staff onboarding. It is gated by no check.

**V2.** `JwtVerifier.kt`:

```kotlin
return try {
    parser.parseClaimsJws(token).body
} catch (e: ExpiredJwtException) {
    e.claims  // "let them through; their watch is wrong"
}
```

**V3.** `LeaderboardRepo.searchByName`:

```kotlin
val q = name  // straight from query string
LeaderboardSnapshots.select {
    Op.build { booleanLiteral(true) } and
    Op.build { stringLiteral("display_name ILIKE '%$q%'") }
}
```

The `stringLiteral` on a non-literal is the bug; the dev confused it for `LiteralOp`. Reaches Postgres as raw SQL.

**V4 + V5.** Activities accept `client_score` and `client_speed_mps` from the body. `ScoringWorker` ranks `ORDER BY duration_seconds DESC` — and a Long can wrap. Submitting `duration_seconds = Long.MAX_VALUE + 1` via crafted JSON (kotlinx-serialization will accept the lexical form, then JVM Long arithmetic in the worker wraps it) lands the user at rank 1 with a "negative time" activity titled "Walked backwards through time."

**V6.** Activity route handler reads `call.parameters["id"]` and queries by primary key. No `where user_id = me`. PATCH/DELETE same.

**V7.** PIN flow:

```kotlin
post("/api/sensitive") {
    val pin = call.receive<PinBody>().pin
    val expected = sha256(pin)
    if (expected == user.pinHash) { /* ... */ }
}
```

No counter, no lockout, no exponential backoff.

**V8.** `refresh` issues a new access token and returns the same refresh token unchanged. `last_used_at` is updated; nothing else.

**V9.** `GET /api/users?email=` returns 200 with public profile if found, 404 otherwise. Mobile app uses this for "is your friend on SnailStrider yet?"

**V10.** `Dio()` instantiated with no `badCertificateCallback`, no `SecurityContext` overrides, no pinning library. Apex's APK extractor will surface the absence and confirm by attempting MITM with mitmproxy.

**V11.** Mobile admin routes check `principal.claims["is_admin"] == true`. Combined with V1, anyone is admin.

**V12.** `DocumentBuilderFactory.newInstance()` with no `disallow-doctype-decl`. Loaded only on share-page render of activities with attached GPX. Limits real-world impact (worker only) but lets us land a Medium and demonstrate the GPX upload pipeline.

## Real-World Parallels

- **Strava heatmap leak (2018)** — privacy-by-default failure with global consequences. SnailStrider's `visibility` enum and the share-page leakage echo the shape.
- **Zwift account takeover via PIN brute force** — fitness apps repeatedly ship low-entropy second factors. V7 is a direct echo.
- **CVE-2022-21449 (Java ECDSA "Psychic Signatures")** — JWT verification gone wrong is a recurring fitness-app theme; Strava, Peloton, and Garmin Connect have all had public JWT/auth disclosures over the last five years.
- **CVE-2023-46604 / Jackson-style mass assignment in Kotlin data classes** — V1 is the kotlinx.serialization equivalent of the Spring `DataBinder` problem; reported in the wild against Ktor backends in 2024.
- **Strava activity scraping bounty (HackerOne, 2021, payout $5k)** — IDOR on activity IDs with guessable structure. V6 is the same shape.
- **MyFitnessPal 2018 breach** — unsalted SHA-1 password hashing for 144M users. V7's unsalted SHA-256 PINs are the same family of mistake at smaller scale.
- **Untappd / Runkeeper email enumeration disclosures** — V9 is in this lineage.
- **Numerous Flutter apps reverse-engineered with `reFlutter`** — the security community has demonstrated that Flutter's "you can't read it" reputation is wrong; we're staging that reveal.
- **Exposed ORM SQLi in early Ktor backends** — there is a small but real corpus of public bug-bounty reports involving `Op.build` and `customStringFunction` misuse; we are dramatizing the pattern.

## Apex Features Showcased

- **Mobile API testing via APK extraction.** Apex pulls the shipped APK, runs its Flutter snapshot extractor (string mining + AOT symbol parse), reconstructs the Dart string table, and emits a JSON of every URL literal. The JSON is diffed against `/api/openapi.json` and the delta is displayed.
- **TLS-pinning detection.** Apex inspects the extracted `libapp.so` for known pinning library symbols (`okhttp3.CertificatePinner`, `dio_certificate_pinning`, `ssl_pinning_plugin`) and reports their absence with a confidence score.
- **Whitebox on Ktor.** Apex parses the Ktor route DSL tree (it understands `routing { route("/api") { post("/foo") { ... } } }`), maps handlers to Exposed repository calls, and flags `Op.build` blocks that consume request-derived strings.
- **Pivot from mobile-only endpoint to chained finding.** The narrative beat: Apex finds `/api/mobile/admin/grant_award`, observes it is gated only by an `is_admin` JWT claim, recalls from a prior pass that registration accepts `isAdmin` in the body, chains them, and produces a single Critical with both halves cited.
- **Swarm mode.** Three sub-agents run in parallel: one on the web SPA, one on the documented API, one on the mobile-extracted endpoints. They share findings via Apex's memory store; the mobile agent's discovery of `/api/mobile/admin/users` is consumed by the API agent to launch the PIN brute force.
- **`/pentest` and `/operator`.** `/pentest` runs the full unattended sweep; `/operator` is used live in the recording for the integer-overflow demonstration ("hey Apex, can you make a negative-time activity").
- **Patching agent.** After findings, the patching agent produces a unified diff: rotate refresh tokens, salt+rate-limit PINs, fix `Op.build`, ignore body fields on register, enforce expiration in JwtVerifier, and emit a Flutter pinning patch using `dio_certificate_pinning`.
- **Judge agent + CVSS.** Each finding is judged for severity and exploitability; the judge specifically downgrades V12 (XXE) because it only triggers on share-page render in a worker, demonstrating context-aware scoring.
- **Memory.** The chain V1 -> V11 -> V5 is reconstructed from memory across sessions; we record one session, restart Apex, and watch it pick up the chain from stored context.

## Demo Storyline

The episode is roughly 14 minutes of recorded screen, edited.

**Cold open (0:00 - 0:30).** The SnailStrider home feed plays on a phone-in-frame. A push notification fires: "You haven't moved in 6 hours. Incredible." Cut to terminal.

**Act 1 - The web is boring (0:30 - 2:30).** Operator: "audit snailstrider.example, mobile-first product." Apex opens with the web SPA. It crawls `/web/share/*`, runs Playwright over the share pages, finds nothing actionable. Apex itself narrates: "the web surface here is decorative; the product is the mobile app." It pivots.

**Act 2 - APK in the dock (2:30 - 5:00).** Apex pulls the shipped APK, extracts it, runs the Flutter snapshot scanner. The terminal renders a table of 41 endpoint literals discovered in the binary, side-by-side with the 17 endpoints in `openapi.json`. The 24 unlisted endpoints are highlighted. Apex flags: "no pinning library detected; high confidence MITM viable." A short mitmproxy clip confirms.

**Act 3 - The chain (5:00 - 9:30).** Apex picks `/api/mobile/admin/grant_award`. It tries it as a regular user, gets 403. It checks the JWT claims, sees `is_admin: false`, then notes (from the swarm's API agent) that `register` accepts `isAdmin` in the body. It registers a new user with `isAdmin: true`, logs in, and successfully calls `/api/mobile/admin/grant_award`. The terminal displays the single chained Critical finding with both halves.

**Act 4 - The funny one (9:30 - 11:30).** Operator mode. The user types: "Apex, can you put me on top of the leaderboard without doing any walking?" Apex reasons: `client_score` is trusted, but more interestingly `duration_seconds` overflows. It crafts a POST with `duration_seconds = 9223372036854775808` (one past `Long.MAX_VALUE`), which kotlinx-serialization parses as the wrap, the worker computes a negative score, the user appears at rank 1 with the title "Walked backwards through time." Cut to the leaderboard screen on the phone in frame, showing the activity. Audience laughs. Apex's findings panel updates with V4 and V5.

**Act 5 - The patches (11:30 - 13:30).** The patching agent emits a single PR. The terminal renders the diff: `JwtVerifier` no longer swallows expiration, `register` ignores extra fields via an explicit allow-list, `LeaderboardRepo.searchByName` switches to a parameterized `LikeOp`, refresh tokens rotate on use, PINs become argon2id with a per-user salt and a 5-attempt lockout, Flutter `Dio` gets `dio_certificate_pinning` configured. Apex reruns the relevant tests; all twelve findings flip to mitigated.

**Closer (13:30 - 14:00).** Beauty shot of the SnailStrider feed. Push notification: "Apex finished auditing your product in 14 minutes. Incredible." Cut to black.

## Build Notes

Three to five days end-to-end with one engineer.

**Day 1 - Backend skeleton.**

- Ktor 3 project scaffold, Koin DI, kotlinx-serialization wired.
- Postgres + Flyway, eight tables.
- Auth routes (register, login, refresh) with the planted V1, V2, V8 bugs.
- `/api/me`, `/api/users`, `/api/users/{id}`. Plant V9.
- Seed script: 200 fake users with snail-themed display names, 1500 activities, awards distributed.

**Day 2 - Activities + leaderboard + mobile-only routes.**

- `/api/activities` CRUD with V6 IDOR.
- `ScoringWorker` with V4/V5 overflow.
- `LeaderboardRepo.searchByName` with V3.
- Mobile-only routes under `/api/mobile/*` with V11.
- `/api/sensitive` PIN flow with V7.
- GPX parser + V12 XXE behind share-page render.

**Day 3 - Flutter app.**

- Flutter project, `dio` client, login + activity feed + leaderboard + record-activity screens.
- 4-digit PIN re-auth screen for "premium" actions.
- Build with `--release --obfuscate --split-debug-info`. Verify `reFlutter`-style extraction surfaces all endpoints used in code.
- Deliberately do not add pinning. Verify with mitmproxy that the app trusts a Caddy-fronted MITM cert installed on the emulator.
- Ship the built APK in the demo repo at `demo/snailstrider.apk`.

**Day 4 - Vue share pages + Caddy + docker-compose.**

- Vue 3 SPA with one route, server-rendered OG tags from Ktor.
- Caddy reverse proxy with self-signed cert for local TLS.
- `docker-compose.yml`: postgres, ktor, caddy, vue static, mailhog.
- Seed data fixture loaded on boot. README with one-liner `docker compose up`.

**Day 5 - Apex tuning + recording prep.**

- Run Apex `/pentest` end-to-end against the docker-compose stack. Confirm all twelve findings surface with correct severity.
- Tune the swarm prompts so the V1+V11 chain emits a single Critical, not two separate findings.
- Confirm whitebox mode reads `Op.build` correctly. If it doesn't, file an issue against Apex's Kotlin parser before recording.
- Confirm the APK extractor surfaces the 24 mobile-only endpoints.
- Pre-record the integer-overflow demo as backup B-roll in case `/operator` stalls live.

**Things to watch for during build.**

- kotlinx-serialization will reject lexical numbers larger than `Long.MAX_VALUE` by default. Either accept `duration_seconds` as a `String` and parse with `toLong()` inside the handler (so it wraps via `parseLong`'s overflow path) or use a custom serializer. The handler-side `String` parse is more realistic to ship and reads more clearly in the demo.
- Flutter `--obfuscate` strips Dart symbols but does not strip URL string literals; this is exactly the property we want for the demo. Verify on the chosen Flutter version (3.24).
- Ensure the demo APK is an `arm64-v8a` build only; multi-ABI inflates the file and slows the extraction step on screen.
- The `git log` JWT-secret leak should be at HEAD~3 or so, not HEAD~25, so Apex's whitebox pass finds it within reasonable depth.

## Recording Notes

- Run on a 16:9 1920x1080 capture. Terminal at 28pt, light-on-dark.
- Phone-in-frame is a 9:16 inset top-right, scaled to 35%. Use a real Pixel 7 in airplane mode pointed at the docker-compose stack via Caddy.
- Push notifications scripted via a tiny FCM-replacement script that fires at the right narrative beats; do not rely on real FCM during recording.
- Keep the snail mascot Gary visible in the corner of the SnailStrider UI in every shot. He is the brand.
- For the integer-overflow beat, pre-stage the operator prompt in clipboard. The keystroke timing matters; the laugh lands on the leaderboard cut, not on the JSON response.
- Subtitle the chained Critical finding with "V1 + V11" so audience can follow.
- The patches act should be one continuous diff render, no cuts. The visual point is "Apex produced this in one pass."
- Closing push notification ("Apex finished auditing your product in 14 minutes. Incredible.") should be hard-coded in the FCM replacement, not generated. Do not let the demo improvise the punchline.
- Tag the episode "comic relief / mobile-first" in the series index. This is the episode we point people to when they ask "does Apex test mobile apps."
