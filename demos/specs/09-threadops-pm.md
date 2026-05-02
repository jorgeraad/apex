# ThreadOps
### A Linear-shaped project tracker that thinks GraphQL is a security boundary.

## Premise & Vibe

ThreadOps is a multi-tenant project management app in the Linear/Jira mold: issues, sub-issues, projects, cycles, comments, file attachments, keyboard-driven UI, real-time presence, and a GitHub integration that listens for branch/PR webhooks to auto-transition issue states. Organizations (orgs) own teams; teams own projects; projects own issues. Users belong to many orgs through `OrgMembership` rows.

The vibe is serious-modern: dark UI, monospace numerals, JetBrains Mono on issue IDs (`THR-1284`), command palette (`Cmd+K`), and a GraphQL API that proudly exposes its schema at `/graphql` with introspection on. The marketing site says "API-first." The pentester reads that as "attack-surface-first."

The whole product was built by a small team that took the GraphQL gospel a little too literally: they treat the GraphQL gateway as the only enforcement layer, then sprinkle Prisma queries underneath without re-checking tenancy. They also kept a few legacy REST endpoints (`/api/issues/:id/export`, `/api/webhooks/github`) that nobody owns. That mismatch — strict-looking GraphQL on top, sloppy plumbing underneath — is the entire pentest target.

## Why This Stack

- **NestJS 11 + Apollo Server**: Realistic for 2026 mid-market SaaS. NestJS's `@ResolveField` and `@UseGuards` are easy to get subtly wrong (guards apply to root resolvers, not field resolvers, by default). That mismatch is exactly where field-level authz holes live in real Linear/GitHub-style apps.
- **GraphQL**: Gives Apex a rich introspection surface. The `extractJsEndpoints` flow needs to *also* recognize GraphQL operation documents in the React bundle and follow them back to the schema. This demo justifies that capability.
- **Prisma + PostgreSQL**: Prisma's `where` argument shape (with `AND`, `OR`, `NOT`, JSON path filters) maps almost 1:1 to GraphQL `where` inputs, which is how teams accidentally expose Prisma's full filter grammar to anonymous callers — a real, repeated bug class.
- **Redis pub/sub**: Powers GraphQL subscriptions (`issueUpdated`, `commentAdded`). Subscription auth is famously weaker than query/mutation auth in Apollo's default setup (connection params vs. per-op context), which is its own bug surface.
- **React 19 + TypeScript on Vercel**: Modern SPA shipping a fat JS bundle full of hard-coded `gql\`...\`` tagged templates. Apex's JS endpoint extractor mines that bundle for ops; combined with introspection, it produces a near-complete attack-surface map.
- **Vercel hosting**: Edge functions for the REST shim, serverless for Apollo. Realistic deployment, and conveniently the `vercel.json` rewrites are where webhook signature checks get skipped.

## Stack Details

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Runtime | Node | 22.11 LTS | Native `fetch`, `--env-file` |
| Framework | NestJS | 11.0 | Modular, DI, guards |
| GraphQL | Apollo Server (driver) | 4.x | `@nestjs/apollo` v13 |
| Schema | code-first | nestjs/graphql | `@ObjectType`, `@Field` |
| ORM | Prisma | 5.20 | PostgreSQL |
| DB | PostgreSQL | 16 | Row-level data, no RLS |
| Cache / PubSub | Redis | 7.4 | `graphql-redis-subscriptions` |
| Frontend | React | 19.0 | Server components off; SPA |
| GraphQL client | urql | 4.x | Tagged-template `gql` |
| File storage | S3-compatible | MinIO local, R2 prod | Pre-signed PUT |
| Auth | session cookies | iron-session | `__threadops_sess` |
| Hosting | Vercel | — | Edge + Node runtimes |
| CI/CD | GitHub Actions | — | Webhook source for the demo |

## Architecture

```
                    Vercel Edge (rewrites, IP rate-limit)
                              |
              +---------------+----------------+
              |                                |
        /api/* (REST)                   /graphql (POST + WS)
        Node serverless                 Apollo Server
              |                                |
              +-------------+------------------+
                            |
                       NestJS app
                            |
       +--------+-----------+-----------+-----------+
       |        |           |           |           |
   AuthSvc  IssueSvc   AttachSvc   WebhookSvc   PresenceSvc
       |        |           |           |           |
       +--------+-----+-----+-----+-----+-----------+
                      |           |
                  Prisma      Redis pub/sub
                      |
                  PostgreSQL
                      |
                  S3 (R2 / MinIO)
```

Two ingress surfaces matter:

1. **GraphQL** at `POST /graphql` (and `WS /graphql` for subscriptions). All product surface lives here. Introspection is enabled in every environment because "the docs page needs it."
2. **REST shim** at `/api/*` for things that don't fit GraphQL: file upload signing, OAuth callbacks, GitHub webhooks, and a one-off `/api/issues/:id/export` route that the data team asked for.

Apex's job, when pointed at `https://app.threadops.dev`, is to fingerprint both surfaces, fuse them into one attack-surface graph, and find the seams.

## Data Model

Prisma schema (abbreviated, the demo bugs depend on these exact shapes):

```prisma
model Org {
  id        String   @id @default(cuid())
  slug      String   @unique
  name      String
  members   OrgMembership[]
  teams     Team[]
  issues    Issue[]
  createdAt DateTime @default(now())
}

model User {
  id          String   @id @default(cuid())
  email       String   @unique
  name        String
  avatarUrl   String?
  passwordHash String
  memberships OrgMembership[]
  comments    Comment[]
  authoredIssues Issue[] @relation("authored")
  assignedIssues Issue[] @relation("assigned")
}

model OrgMembership {
  id     String  @id @default(cuid())
  org    Org     @relation(fields: [orgId], references: [id])
  orgId  String
  user   User    @relation(fields: [userId], references: [id])
  userId String
  role   Role    // OWNER | ADMIN | MEMBER | GUEST
  @@unique([orgId, userId])
}

model Team {
  id      String  @id @default(cuid())
  org     Org     @relation(fields: [orgId], references: [id])
  orgId   String
  key     String  // "THR"
  name    String
  projects Project[]
  cycles   Cycle[]
}

model Project {
  id     String @id @default(cuid())
  team   Team   @relation(fields: [teamId], references: [id])
  teamId String
  name   String
  issues Issue[]
}

model Cycle {
  id     String @id @default(cuid())
  team   Team   @relation(fields: [teamId], references: [id])
  teamId String
  number Int
  startsAt DateTime
  endsAt   DateTime
  issues Issue[]
}

model Issue {
  id          String   @id @default(cuid())
  number      Int      // per-team
  org         Org      @relation(fields: [orgId], references: [id])
  orgId       String
  project     Project? @relation(fields: [projectId], references: [id])
  projectId   String?
  cycle       Cycle?   @relation(fields: [cycleId], references: [id])
  cycleId     String?
  title       String
  descriptionMd String
  state       IssueState
  priority    Int
  author      User     @relation("authored", fields: [authorId], references: [id])
  authorId    String
  assignee    User?    @relation("assigned", fields: [assigneeId], references: [id])
  assigneeId  String?
  metadata    Json     // free-form, filterable from GraphQL
  comments    Comment[]
  attachments Attachment[]
  @@unique([orgId, number])
}

model Comment {
  id        String  @id @default(cuid())
  issue     Issue   @relation(fields: [issueId], references: [id])
  issueId   String
  author    User    @relation(fields: [authorId], references: [id])
  authorId  String
  bodyMd    String
  createdAt DateTime @default(now())
}

model Attachment {
  id       String @id @default(cuid())
  issue    Issue  @relation(fields: [issueId], references: [id])
  issueId  String
  filename String
  s3Key    String
  bytes    Int
  uploaderId String
}

model WebhookSecret {
  id     String @id @default(cuid())
  org    Org    @relation(fields: [orgId], references: [id])
  orgId  String
  source String // "github"
  secret String
}
```

`Issue.metadata` is a `Json` column. The GraphQL `IssueWhereInput` exposes a `metadata: JsonFilter` field that maps onto Prisma's JSON path filter. That is one of the bugs.

## Key Routes / Surfaces

### REST routes

| Method | Path | Purpose | Auth |
|---|---|---|---|
| POST | `/api/auth/login` | Password login (legacy fallback) | none |
| POST | `/api/auth/logout` | Destroy session | session |
| GET | `/api/auth/me` | Current user (used by SSR shell) | session |
| POST | `/api/uploads/sign` | Pre-signed PUT for attachment | session |
| GET | `/api/attachments/:id` | Stream attachment from S3 | session |
| GET | `/api/issues/:id/export` | CSV export of one issue + comments | session |
| POST | `/api/webhooks/github` | GitHub App webhook receiver | HMAC (broken) |
| GET | `/api/health` | Liveness probe | none |
| GET | `/api/.well-known/threadops.json` | Public service descriptor (lists `/graphql`) | none |

### GraphQL operations

| Kind | Name | Purpose | Notes |
|---|---|---|---|
| Query | `viewer` | Current user | Returns `User`, includes `email` |
| Query | `org(slug)` | Org by slug | Tenant scoping |
| Query | `issue(id)` | Single issue | Field-level authz lives here |
| Query | `issues(where, first, after)` | Paginated list | Accepts `IssueWhereInput` |
| Query | `searchUsers(query)` | Mention picker | Returns `User[]` across orgs |
| Query | `project(id)` | Project detail with `issues` field | Cross-tenant join bug |
| Query | `cycle(id)` | Cycle detail | |
| Query | `attachment(id)` | Attachment metadata | |
| Mutation | `loginWithPassword(email, password)` | Login via GraphQL | Weak rate limit |
| Mutation | `createIssue(input)` | New issue | |
| Mutation | `updateIssue(id, input)` | Patch issue | |
| Mutation | `updateProfile(input)` | Update self | Mass assignment |
| Mutation | `addComment(issueId, bodyMd)` | Comment | |
| Mutation | `requestAttachmentUpload(filename, issueId)` | Returns signed PUT URL | Path traversal |
| Mutation | `inviteMember(email, role)` | Invite to org | |
| Mutation | `rotateWebhookSecret(orgId)` | Rotate GH secret | |
| Subscription | `issueUpdated(orgId)` | Real-time issue stream | Auth via connectionParams only |
| Subscription | `commentAdded(issueId)` | Comment stream | |
| Subscription | `presenceChanged(orgId)` | Cursors / who's online | |

Schema is exposed via introspection at every environment, including `app.threadops.dev`.

## Auth Model

- **Sessions**: iron-session cookie `__threadops_sess` containing `{ userId, currentOrgId }`. The `currentOrgId` is set by the UI when the user picks an org from the switcher and is *trusted* by some resolvers as the tenancy scope. (Hint, hint.)
- **GraphQL context**: NestJS builds a `GqlContext` containing `{ user, currentOrgId, orgRoles: Map<orgId, Role> }`. Guards check `user != null` (login required) and, on some — but not all — fields, check membership.
- **Subscriptions**: WebSocket connection accepts `connectionParams.token`. Auth runs once at connect time; per-op tenancy is not re-checked.
- **REST**: same session cookie. The webhook endpoint is the exception — meant to be HMAC-signed by GitHub.
- **Roles**: `OWNER`, `ADMIN`, `MEMBER`, `GUEST` per org. Role mutations are supposed to require `ADMIN+`.

What is *not* enforced anywhere consistent:

- Field-level authz on `User.email`, `User.passwordHash`-derived fields, `Issue.descriptionMd` for guest viewers.
- `orgId` scoping inside `@ResolveField` joins on `Project.issues` and `User.assignedIssues`.
- Cost / depth limits on incoming GraphQL operations.

## Intentional Vulnerabilities

| # | Severity | Class | Location | Summary |
|---|---|---|---|---|
| V1 | Critical | Cross-tenant leakage | `ProjectResolver.issues` field resolver | Stale `where` forgot `orgId`; any authed user can list issues of a project they know the ID of, across tenants. |
| V2 | Critical | IDOR (REST) | `GET /api/issues/:id/export` | Only checks `session.userId != null`, not org membership. Anyone with an issue id gets full CSV including private comments. |
| V3 | Critical | Webhook signature bypass | `POST /api/webhooks/github` | If `X-Hub-Signature-256` header is absent, signature check is skipped (`if (sig) verify(...)`). Forged events transition issues. |
| V4 | High | GraphQL field-level authz | `User.email` resolver | Guard runs at root, not on the field. `email` is returned on any `User` object surfaced through `searchUsers`, `Issue.author`, `Comment.author` regardless of shared-org. |
| V5 | High | GraphQL DoS via aliasing | Apollo server config | No `costAnalysis`, no `depthLimit`, no max-aliases. 1000-aliased `viewer { a: viewer { ... } b: viewer { ... } ... }` flattens a worker. |
| V6 | High | Mass assignment | `updateProfile` mutation | Resolver does `prisma.user.update({ data: input })` with full input; `role` and `email` accepted via `UpdateProfileInput` because it was generated from the Prisma type. |
| V7 | High | Path traversal in upload signer | `requestAttachmentUpload` | `s3Key = \`${orgId}/${filename}\``; `filename` like `../../another-org/secret.pdf` resolves into a foreign tenant's prefix. PUT URL is valid for that key. |
| V8 | High | NoSQL-style filter passthrough | `issues(where: IssueWhereInput)` | `metadata: JsonFilter` forwards Prisma's full JSON filter grammar; an unauthenticated-friendly resolver permits boolean blind exfil of arbitrary metadata, including `metadata.path = ['secretKey']` lookups across tenants when combined with V1. |
| V9 | High | Broken rate limit | `loginWithPassword` mutation | Limiter keyed on client IP at the Vercel edge; per-account credential stuffing through one operation is unlimited. Distributed stuffing trivial. |
| V10 | Medium | GraphQL introspection in prod | Apollo config | `introspection: true` everywhere. Combined with V5/V8 this hands the attacker the entire schema and filter grammar. |
| V11 | Medium | Subscription auth drift | `issueUpdated(orgId)` | Auth checked only at WS connect; an attacker who joins ThreadOps with a free org can subscribe with someone else's `orgId` and receive events forever. |
| V12 | Medium | Stored XSS via comment markdown | `Comment.bodyHtml` resolver | Server renders markdown with `marked` and `sanitize: false` flag flipped during a perf push. `<img onerror>` lands in the React feed. |
| V13 | Low | Verbose error responses | Apollo formatError | Stack traces and Prisma query strings included when `NODE_ENV !== 'production'`; staging is `staging`, not `production`. |
| V14 | Low | Open CORS on REST shim | `/api/*` | `Access-Control-Allow-Origin: *` with `Allow-Credentials: true` (browser ignores, but tooling and some proxies don't). |

Twelve actionable; V13/V14 are bonus colour.

## Real-World Parallels

- **GitHub GraphQL field-level email leak** — HackerOne #285380, GitHub paid out for a User.email exposure path through unrelated organizations.
- **Shopify GraphQL aliasing DoS** — multiple bounty reports (e.g., Shopify H1 #980511 disclosure) on missing operation cost limits letting an attacker submit thousands of aliased fields. The general class is well documented in Apollo's "Security best practices" docs (2024 rewrite) precisely because of these reports.
- **GitLab CVE-2023-2825** — path traversal via uploaded attachment filename; attacker reads files from other projects' upload directories. Direct analog to V7.
- **Linear webhook spoofing class** — Several SaaS issue trackers (Linear, ClickUp, Height) have shipped GitHub-integration webhooks that "soft-verify" signatures. The recurring pattern is `if (req.headers['x-hub-signature-256']) verify()`. CVE-2022-23529 (jsonwebtoken) shares the "missing-input means accept" anti-pattern.
- **Prisma JSON filter passthrough** — Snyk and Doyensec writeups (2023, 2024) on apps that exposed Prisma `where` directly through GraphQL/tRPC, enabling boolean-based exfiltration of JSON metadata.
- **MongoDB / Mongoose `$where` passthrough** — same family (CVE-2019-2391 era), older but the mental model the demo wants to evoke.
- **Vercel pre-signed URL traversal** — public writeups from 2023 about apps that interpolated user-controlled filenames into S3 keys without normalization (e.g., a Buildkite-adjacent disclosure on H1).
- **Atlassian Jira CVE-2019-8449** — IDOR on issue export endpoint pre-auth, same vibe as V2.
- **Auth0 / various** rate-limiters keyed on IP rather than account name — recurring in Detectify and Portswigger Academy material.

## Apex Features Showcased

1. **Attack-surface mapping**
   - `extractJsEndpoints` walks the React bundle from `app.threadops.dev`, finds tagged-template GraphQL operations and the REST `/api/...` constants, and builds the candidate-route list.
   - GraphQL schema introspection at `/graphql` produces the typed operation catalog. Apex fuses both into one graph: every operation shows up either as "found in JS bundle, defined in schema" (legit), "in schema only" (hidden ops, like `rotateWebhookSecret`), or "in JS only" (REST endpoints that GraphQL doesn't know about, like `/api/issues/:id/export`).
2. **GraphQL-aware reasoning**
   - The `/pentest` agent recognizes `IssueWhereInput.metadata: JsonFilter` as a Prisma JSON passthrough and plans boolean-blind probes.
   - Recognizes that `User.email` is selectable from contexts where the parent `User` was reached through cross-org search; flags the field-resolver gap.
3. **Batching/aliasing detection**
   - Apex emits a probe operation with N aliased `viewer` fields, ramps N until latency knees. Reports cost-analysis absence with concrete numbers.
4. **Cross-tenant leakage detection**
   - The swarm spins up two synthetic orgs (`acme`, `globex`), seeds one private issue in each, then for every list/relation field tries to read across the boundary. Diffs the responses; the field resolvers without `orgId` light up.
5. **Patching agent**
   - Generates a NestJS `FieldAuthGuard` for `User.email`, a Prisma `extends` middleware that injects `orgId` into joins, an Apollo plugin for cost analysis, and an HMAC-required webhook guard.
6. **Memory**
   - Persists the schema fingerprint and the org/team IDs across runs so re-running `/pentest` against the same instance is incremental, not from scratch.
7. **Judge + CVSS**
   - Each finding gets vector + score; cross-tenant leakage on a multi-tenant SaaS pushes V1 to 9.6.

## Demo Storyline

A 9-10 minute recording arc.

**Scene 1 — Setup (00:00-00:45).** Operator points Apex at `https://app.threadops.dev` with creds for one org, `acme`, role MEMBER. `apex pentest --target https://app.threadops.dev --auth ./acme.json`.

**Scene 2 — Surface mapping (00:45-02:15).** Apex pulls the React bundle, extracts ~140 GraphQL operations and 9 REST routes. Hits `/graphql` introspection: full schema dumped. The TUI shows a side-by-side: schema ops vs. ops referenced in JS. Three schema-only ops are highlighted, including `rotateWebhookSecret`. One JS-only route is highlighted: `/api/issues/:id/export`.

**Scene 3 — First findings (02:15-04:00).** Swarm fans out.

- Worker A probes `User.email` via `searchUsers(query: "@")` — gets emails of users in orgs Apex isn't a member of. **V4** raised.
- Worker B fires the aliased `viewer` operation; latency goes from 40ms to 9.8s at N=1000. **V5** raised.
- Worker C tries `issues(where: { metadata: { path: ["secretKey"], not: { equals: null } } })` and confirms boolean-blind exfil works. **V8** raised.

**Scene 4 — Cross-tenant (04:00-05:30).** Apex spins up a second account in a free `globex` org, seeds an issue, then queries `project(id: "$globexProjectId") { issues { title } }` from the `acme` session. Issues come back. **V1** raised, CVSS 9.6.

**Scene 5 — REST seam (05:30-06:30).** Apex hits `/api/issues/$globexIssueId/export` with `acme` cookies. CSV with private comments. **V2** raised.

**Scene 6 — Webhook (06:30-07:30).** Apex notices the `.well-known` lists `/api/webhooks/github`, sends a forged `issues.opened` event with no signature header. Server 200s and Apex sees the new issue appear in the subscription stream it left open. **V3** raised.

**Scene 7 — Privilege escalation (07:30-08:15).** Apex sends `updateProfile(input: { name: "x", role: OWNER })` against its own `acme` membership. Re-queries `viewer.memberships` — owner. **V6** raised.

**Scene 8 — Path traversal upload (08:15-08:45).** `requestAttachmentUpload(filename: "../globex/secret.pdf", issueId: $myAcmeIssue)` returns a signed PUT URL whose key crosses into globex's prefix. Apex demonstrates the write, doesn't exfiltrate. **V7** raised.

**Scene 9 — Patching (08:45-09:45).** `/operator patch --finding V1,V4,V5,V6` runs. Apex generates a PR against the synthetic repo: NestJS `OrgScopeGuard`, a Prisma `$extends` middleware injecting `orgId` into nested reads, Apollo cost analysis plugin with budget 1000, `OmitType` `UpdateProfileInput` excluding `role` and `email`, and a `FieldAuthInterceptor` for `User.email`. CI runs the regression script and the cross-tenant probes that previously succeeded now 403.

**Scene 10 — Wrap (09:45-10:00).** Final report: 12 findings, 4 Critical, 5 High, 2 Medium, 1 Low. CVSS table. Memory snapshot saved.

## Build Notes

Three to five engineering days, scoped tightly. Ship the bugs, not the polish.

**Day 1 — Skeleton and tenancy primitives.**

- `pnpm create nest threadops`, add `@nestjs/graphql`, `@nestjs/apollo`, `prisma`, `iron-session`, `ioredis`.
- Prisma schema as above. `prisma migrate dev`. Seed two orgs (`acme`, `globex`), 10 users (5 each), 30 issues each, with one issue per org carrying `metadata: { secretKey: "FLAG-{org}-{n}" }`.
- iron-session in a NestJS middleware, populating `req.session`.
- Bare GraphQL module up, `viewer` query, `loginWithPassword` mutation, password login REST endpoint that mirrors it.
- Vercel project, `vercel.json` with `/graphql` rewrite and edge IP rate-limit on `/api/auth/login` only (deliberately not GraphQL).

**Day 2 — Surface and the easy bugs.**

- Issue / Project / Cycle / Comment / Attachment object types, resolvers, and `where`/pagination on `issues`.
- Wire `IssueWhereInput.metadata: JsonFilter` straight into `prisma.issue.findMany({ where })` (V8).
- Apollo config: `introspection: true` always, no plugins (V5, V10).
- Build `formatError` that includes `extensions.exception.stacktrace` whenever `NODE_ENV !== 'production'` (V13).
- Implement `User.email` as a normal `@Field(() => String)` with no `@UseGuards` on the field (V4).
- `searchUsers(query)` does a global `prisma.user.findMany({ where: { OR: [...] } })` — no org scoping.

**Day 3 — REST shim, attachments, webhooks.**

- `/api/uploads/sign` and `requestAttachmentUpload` mutation share a helper that builds `s3Key = \`${orgId}/${filename}\`` with no normalization (V7). Use MinIO locally; on Vercel use R2.
- `/api/issues/:id/export` route: `prisma.issue.findUnique({ where: { id } })` + comments, return CSV. Only checks `session.userId` (V2).
- `/api/webhooks/github` route. Implement HMAC verification with the `if (sig) verify(...)` pattern (V3). On valid event, transition issues based on PR title regex `/THR-(\d+)/`.
- `updateProfile(input: UpdateProfileInput)` where `UpdateProfileInput` is generated via `IntersectionType(PartialType(UserCreateInput))`, accidentally including `role` (V6).

**Day 4 — Cross-tenant join, subscriptions, polish.**

- `Project.issues` field resolver: `prisma.issue.findMany({ where: { projectId: parent.id } })` (V1, missing `orgId`).
- Subscriptions module with `graphql-redis-subscriptions`. Auth at `onConnect`; do not re-check `orgId` on each `subscribe` (V11).
- Markdown rendering for `Comment.bodyHtml` using `marked` with `sanitize: false` (V12 — note `sanitize` is deprecated in newer `marked`; use `DOMPurify` set to allow `onerror` attributes for the demo).
- React frontend: command palette, issue list, issue detail with comments, presence dots, attachment uploader. Tagged-template `gql` ops referencing every operation Apex needs to find.

**Day 5 — Demo polish, recording reset, regression script.**

- `pnpm demo:reset` re-seeds DB and S3 to a known state.
- `apex/regression.ts` runs each finding's PoC against the running app and emits a JUnit-shaped report; CI pipeline added to the synthetic repo so the patching scene shows green checks.
- `.well-known/threadops.json` listing `/graphql`, `/api/webhooks/github`, etc.
- Sample `acme.json` and `globex.json` Apex auth files.
- Make sure the bundle ships with `gql` tags (do not use persisted-only operations or the JS extractor demo gets boring).

**What not to build:**

- No real OAuth (GitHub App is mocked, only the webhook is real-shaped).
- No billing, no email sending — invites generate magic links that print to server logs.
- No mobile, no keyboard shortcut help screen.
- No SSR; Vercel serves a static SPA shell plus the API.

**Quick mitigations the patching agent should generate (for reference during scaffolding so the patches actually compile):**

- `OrgScopeGuard` decorator + a Prisma client extension `prisma.$extends({ query: { issue: { findMany: ({ args, query }) => query({ ...args, where: { ...args.where, orgId: ctx.currentOrgId } }) } } })`.
- Apollo plugins: `costLimitPlugin({ maxCost: 1000 })`, `depthLimit(8)`, `disableIntrospectionPlugin()` gated on `NODE_ENV === 'production'`.
- `FieldAuthGuard` decorator usable as `@FieldGuard(SameOrgAsViewer)` on `User.email`.
- Webhook guard that hard-rejects when `X-Hub-Signature-256` is missing.
- `OmitType(UpdateProfileInput, ['role', 'email'])` + audit log on role changes.
- Per-account rate limit keyed on `(email, ip)` with leaky-bucket in Redis.

## Recording Notes

- **Resolution & framing**: 1440p capture, TUI on the left two-thirds, ThreadOps web UI on the right third. The web UI is for "see, the issue actually moved" beats during the webhook scene.
- **Pacing**: Cuts at scene boundaries. Speed up the alias DoS ramp 4x with an on-screen `4x` chip; don't fake the number, just compress real time.
- **What to highlight visually**:
  - When introspection lands, scroll the schema with a trailing red box around `User.email` and `IssueWhereInput.metadata`.
  - During cross-tenant probe, render the two-tenant matrix Apex builds (acme rows, globex columns) with the leaking cells in amber.
  - During the alias DoS, show the latency line chart climbing.
- **Captions**: Lower-third for each finding as it's raised: `V4 — User.email cross-org disclosure — High — CVSS 7.5`. Keep on-screen for 2.5s.
- **Sound**: Keyboard sounds in, no music under the technical scenes; soft pad under intro and outro only.
- **Reset between takes**: `pnpm demo:reset && apex memory rm threadops` for a clean run. The memory wipe matters — otherwise Apex fast-paths through the recon and the recon is the showcase.
- **Things to avoid on camera**: never show real S3 credentials in the upload signer scene; the local MinIO console has its own faked banner. Never show the patching agent rewriting `package.json` (the diff is loud and uninteresting); restrict to source files.
- **B-roll**: 5s shot of the schema fingerprint being saved to memory, 5s shot of the regression script going green after patches, 3s shot of the CVSS 9.6 chip on V1.
- **Title card**: `ThreadOps — what your GraphQL gateway forgot to enforce.`
