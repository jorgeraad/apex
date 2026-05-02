# Self-Referential Demos — Apex on Apex, Apex on Console

This spec covers the two finale episodes of the Apex demo series. Both
are dogfooding scenarios in which the Apex CLI is pointed at
infrastructure that is part of the Apex / Pensar product surface
itself. Because the target overlaps with our own production code and
our own cloud service, this spec reads more like a tabletop exercise
plan than a marketing storyboard. The two scenarios are specified
together because they share the same risk model: a real critical
finding could surface on camera, in real time, against code we ship or
against a service that holds customer data. The operational disciplines
required — pre-approval, staging-only execution, finding-handling
protocols, editorial review before publish — are common to both.

## Why these demos exist

The series so far has shown Apex working against external, fictional
target apps: a fintech, a healthcare portal, an LMS, an e-commerce
platform, a CI/CD service, an HR system, a government benefits portal,
and a PM tool. Those demos are credible because the targets are
plausible, but they are still simulations. The natural and honest finale
of the series is to turn the tool inward.

Four reasons to ship these episodes, articulated up front in the
editorial brief so production and reviewers share goals.

First, dogfooding. A pentesting tool that has never been pointed at its
own codebase is a tool whose authors have not taken their own product
seriously. If we are not willing to do it, we cannot reasonably ask
customers to do it.

Second, trust. Security tooling is a trust-heavy category. Buyers
discount marketing aggressively and weight observable behavior heavily.
A demo in which we run our own tool against our own code — and ship
the resulting findings, fixes, and post-mortem — is a durable trust
signal that narrative-only content cannot replicate.

Third, transparency. Pointing Apex at Pensar Console (under the caveats
below) communicates that we hold our cloud service to the same standard
we expect customers to hold theirs to. We do not have to publish every
finding. We do have to publish the fact that we ran the exercise.

Fourth, internal training. Building these episodes forces the team to
write a coherent threat model for our own product, keep it current,
maintain a vulnerable fork for training, and rehearse incident response
under low-stakes conditions. The production work is itself the training.

Non-goals: competitive positioning (we are not benchmarking against
other tools), and shock value (we do not promise a critical RCE on
camera; finding nothing exploitable in the production-equivalent run
is a fine outcome and the episode should say so).

## Operating principles for self-referential demos

The following rules apply to both scenarios and override anything in
the per-scenario sections that contradicts them.

1. No live customer data. No demo run touches a database, queue, object
   store, or log stream containing real customer data. For Scenario B
   this means a staging clone with synthetic fixtures only.

2. Staging-only for Console. Scenario B is not run against
   `console.pensar.dev` or any production hostname. A dedicated staging
   environment is stood up for the recording, isolated by network and
   credentials from production.

3. Pre-approval required. Both scenarios require written sign-off from
   engineering leadership, security, and legal. Scenario B additionally
   requires sign-off from whoever owns the staging environment.

4. Vulnerable fork for Scenario A. The on-camera run executes against
   an `apex-target-vulnerable` branch with intentionally introduced
   bugs. This branch is never published as a release. A second,
   off-camera run executes against current `main` for genuine
   dogfooding; those results go through the normal vulnerability
   process and do not appear in the episode unless cleared.

5. Real findings on camera are stop-the-recording events. Credible
   critical or high findings trigger a verbal "hold," embargo of the
   take, and a go/no-go decision by security leadership before further
   takes. This applies whether the finding is against the vulnerable
   fork or current production code.

6. Secrets never reach the cut. The recording environment uses a
   throwaway `~/.pensar/` with synthetic credentials. Real Pensar
   Console credentials are not present on the recording machine. Any
   frame rendering a token, cookie, JWT, env var, or signed URL is
   redacted in post regardless of whether the value is real. A real
   secret on screen destroys the take and rotates the secret.

7. The recording machine is single-purpose. No personal shell history,
   browser profiles, SSH keys, or cloud credentials. Wiped after the
   episode ships.

8. On camera: tool invocations, agent reasoning summaries, findings
   registry view, patching agent diffs, redacted threat model
   documents, editorial commentary.

9. Not on camera: raw network traffic against Console, Playwright
   frames showing internal admin UI, MCP payloads with bearer tokens,
   raw HMAC signatures, cookie values, env var dumps, contents of
   `~/.pensar/` files, `.env` contents, internal Slack/Linear screens,
   internal infrastructure dashboards.

10. Editorial review before publish by security and legal: incidental
    secret leakage, unredacted internal hostnames, accidental customer
    name disclosure in fixtures.

11. Publish-time gating. A finding from the on-camera run that remains
    unfixed at publish time holds the episode until fixed or until
    explicitly cleared for disclosure with a fix advisory attached.

## Scenario A — Apex Pentests Apex

### Premise

Apex is invoked against a checkout of its own source tree. The framing
on camera is "we built a pentesting tool, let's see what it says about
its own codebase." The on-camera target is the
`apex-target-vulnerable` branch — a fork with a curated set of
intentionally introduced, realistic bug classes. The episode discloses
this clearly: the branch contains seeded bugs for demonstration
purposes, and a parallel run executed against current `main` was
handled through the normal vulnerability process.

The episode features whitebox mode and spotlights `/pentest
--threat-model` ingestion, the sub-agent swarm, the findings registry
with CVSS scoring, the judge agent triaging and deduplicating, the
patching agent producing a candidate diff, and persistent memory
across runs.

### Target Surface

In scope (subset of `/home/user/apex`):

- The CLI entry point and slash command dispatcher
- The auth subsystem that performs device flow and HMAC signing for
  Pensar Console
- The MCP bridge layer and any tool-wrapper code that shells out to
  external binaries
- The Playwright sandbox runner and the browser tool surface
- The email tool and any other side-effect-capable tool wrappers
- Config and state handling under `~/.pensar/` (path traversal,
  permissions, parsing)
- The findings registry storage layer
- The CI workflow files (any `secrets.*` references, third-party
  actions, pinning posture)

Out of scope for the on-camera run: the Anthropic SDK and other
vendored SDKs (covered by SCA separately); the TUI rendering layer;
model provider routing logic; anything under `demos/`. The off-camera
run against `main` covers everything without these exclusions.

### Stack & Architecture

TypeScript on the Bun runtime. React Ink for the TUI. Anthropic SDK
plus multi-provider routing for model calls. Auth via Pensar Console
using device-flow OAuth-like exchange and HMAC-signed requests on the
hot path. MCP bridges over stdio and HTTP. Playwright for browser tasks
in a sandboxed subprocess. State on disk under `~/.pensar/`. CI on
GitHub Actions with lint, type-check, test, and build jobs.

### Threat Model

The threat model below is the artifact fed to `/pentest --threat-model
threat-model.md`. It is shown on camera (redacted as needed) so the
audience sees the input the tool reasons over.

Assets:

- User's Pensar Console credentials and HMAC signing key on disk under
  `~/.pensar/auth.json`
- Findings data in `~/.pensar/findings/` — may include sensitive details
  about user's target systems
- Persistent memory in `~/.pensar/memory/` — may include user prompts
  and intermediate model outputs
- The user's shell environment, including arbitrary env vars
- The user's local filesystem as accessible by the CLI process
- Outbound network access from the user's machine
- Any container or VM the user has provisioned for Kali tooling

Trust boundaries:

- CLI process vs. user shell — CLI inherits user privileges
- CLI process vs. spawned subprocess (Playwright, Kali container, MCP
  child) — subprocess-level boundary, exploitable if argv or env is
  attacker-controlled
- CLI process vs. remote MCP server — network boundary, transport auth
  matters
- CLI process vs. Pensar Console — network boundary, HMAC + device token
- CLI process vs. model provider — network boundary, provider-issued
  credentials in env

Adversaries:

1. Malicious target system. The system Apex is pointed at is, by
   definition, untrusted. It can return crafted HTML, headers, JSON,
   filenames, and binary blobs. Anything Apex parses, renders, or
   feeds to a tool is potentially adversarial.
2. Malicious MCP server. A user might configure a third-party MCP
   server. That server can return crafted tool definitions, descriptions
   that contain prompt-injection payloads, and crafted tool outputs.
3. Malicious model output. Prompt injection from any of the above can
   cause the model to emit tool calls with attacker-influenced
   arguments. Tool wrappers must treat model-emitted arguments as
   untrusted input, not as already-validated.
4. Local unprivileged process. Another process on the user's machine,
   running as the same user, that wants to read `~/.pensar/`.
5. Supply-chain. A compromised dependency or a compromised CI step.

Non-adversaries (out of scope for this threat model):

- Kernel-level attackers
- The model provider itself
- Pensar's own infrastructure (covered by Scenario B)

Assumptions:

- The user runs Apex on a machine they control
- The user has not deliberately disabled sandboxing flags
- The Bun runtime and the OS are up to date

### Plausible Findings

The list below describes plausible candidate findings — bug classes that
have historically affected tools with similar shapes. They are not
claims about specific bugs in the current Apex codebase. The
`apex-target-vulnerable` branch will seed a curated subset of these so
the on-camera run has demonstrable hits; the seeded subset is selected
to span categories without telegraphing real production weaknesses.

Categorization legend:
- E = "would be embarrassing if real"
- B = "expected and benign"
- I = "interesting if found"

1. (E) Command injection in a tool wrapper shelling out to a pentest
   binary, where an argument is built by string concatenation from
   model-emitted parameters. Reference class: CLI tools wrapping
   `nmap`, `ffuf`, `sqlmap` have had argv-injection issues repeatedly.

2. (E) Path traversal in code that reads or writes findings under
   `~/.pensar/findings/` using a finding identifier or target hostname
   as a path component (`..` sequences, absolute paths in identifiers).

3. (E) HMAC signing that canonicalizes some but not all of headers /
   query / body, allowing a request signed for one operation to be
   replayed against another. Reference class: SigV4-style canonical-
   ization gaps.

4. (E) Device-flow polling that does not bind the access token to the
   device that initiated the flow. Reference class: OAuth device flow
   implementations that skip `device_code`-to-token binding.

5. (E) MCP transport over HTTP defaulting to no auth, or accepting a
   token via query string where it lands in proxy logs.

6. (E) Playwright sandbox launched with `--no-sandbox` or with a
   user-data-dir under a world-readable path.

7. (E) Prompt-injection-driven tool call where a malicious target page
   causes the agent to write outside the working directory.

8. (E) `~/.pensar/auth.json` written with mode 0644/0666 instead of
   0600, exposing credentials on multi-user systems.

9. (I) TOCTOU between permission check and file open in the findings
   store; exploitable only with a racing local process.

10. (I) Insufficient rate limiting on device-flow polling allowing
    brute-force of short device codes (client-side polling-interval
    enforcement missing).

11. (I) MCP tool descriptions rendered into the model context without
    sanitization, letting a hostile MCP server run prompt injection.

12. (B) Verbose CLI errors with stack traces containing absolute paths.
    Embarrassing-adjacent, generally not exploitable.

13. (B) Outdated transitive dependency with a published advisory but
    no reachable code path. SCA noise.

14. (B) ReDoS in a parser used only against user-typed input.

15. (I) CI workflow running a third-party action pinned to a floating
    tag rather than a commit SHA. Flagged, not exploited on camera.

The on-camera narrative wants two or three (E) hits and one (I) hit.
Anything that lands beyond that is bonus and editorially trimmed. The
seeded bugs in the vulnerable fork are chosen from this list and
documented in a private file (not in this spec) so that we can verify
the agent finds them.

### Recording Strategy

Two passes. Pass one is the genuine run, off camera, against current
`main`; the resulting findings are triaged through the normal process,
and nothing appears on camera unless explicitly cleared. Pass two is
the on-camera run against `apex-target-vulnerable` in a single-purpose
recording VM, with a synthetic `~/.pensar/` and the Pensar Console
endpoint pointed at a local mock returning canned responses, so the
auth surface generates no real traffic.

Real-time secret redaction: terminal recording is post-processed to
mask token, cookie, and signed-URL patterns; the shell prompt is a
single character with no hostname or username; the TUI is configured
to truncate over-length values and mask sensitive-tagged fields; a
designated reviewer mirrors the recording on a second monitor with a
kill switch and watches only for accidental secret rendering.

Handling a real critical finding mid-recording: verbal "hold," camera
stops, take moves to embargoed storage (not deleted; we may need it
for the post-mortem), finding filed under restricted visibility,
go/no-go from security leadership before further takes. No-go pauses
the episode until the issue is resolved.

Cut in post: frames with unredacted token-shaped strings even if
synthetic (viewers do not know they are synthetic and may copy them);
contents of `~/.pensar/` files even if synthetic; agent reasoning
naming internal staff or hostnames; operator typing that reveals
internal paths, names, or identifiers.

### Cleanup Plan

After the episode wraps: (1) recording VM is wiped; (2) pass-one
findings are filed in the internal tracker and tracked through the
normal vulnerability process; (3) pass-two findings are reconciled
against the seeded-bug list to confirm seeded bugs were found and to
flag any unseeded bugs surfaced incidentally — those escalate to
pass-one handling; (4) the `apex-target-vulnerable` branch is preserved
as a reusable training and regression-testing asset, but its bug list
is not public; (5) a post-mortem covers what was found, what was
missed against seeded, what the operator nudged, what the editorial
team cut, and what we will change in the product; (6) patching-agent
diffs shown on camera are either landed (if the bug exists on `main`)
or discarded (if it exists only on the vulnerable fork).

## Scenario B — Apex Pentests Pensar Console

This scenario is described in the conditional. The actual production of
this episode is contingent on approvals that have not been granted at
the time of this spec. Every specific Console subsystem named below is
based on public characterization of the service and on standard SaaS
patterns; treat each named subsystem as an assumption to be validated
with the Console team before this scenario moves to production.

### Premise

Apex is pointed at a staging clone of Pensar Console. The framing on
camera is "we built a pentesting tool that talks to a cloud service
that we also built — let's run the tool against the cloud service." The
episode demonstrates blackbox mode against a real-looking cloud target,
including attack-surface mapping, JS endpoint extraction, and the agent
swarm coordinating discovery.

This episode is the harder of the two to ship. The target is a
multi-tenant SaaS that holds customer data in production. Even with
staging-only execution, the optics of "the vendor pentested its own
cloud service on YouTube" are sensitive and the episode is held to a
higher editorial bar.

### Target Surface

The Console subsystems below are proposed targets, contingent on
confirmation by the Console team that each is in scope and that a
staging clone with synthetic data exists for each.

Likely-in-scope (proposed):

- The auth subsystem — device-flow endpoints, session handling, HMAC
  verification on signed routes, token revocation, rate limiting on
  login and device-code endpoints
- The workspaces API — CRUD on workspace and membership resources,
  authorization checks on cross-workspace operations, IDOR surface on
  resource identifiers
- The agent webhooks — endpoints that accept callbacks from running
  agents, signature verification, replay protection, payload size and
  content-type handling
- The findings sync API — endpoints that the CLI uses to push and pull
  findings, authorization scoping, and tenant isolation
- The public marketing pages and docs — only insofar as they leak
  internal hostnames, expose source maps, or expose admin-adjacent
  endpoints

Likely-out-of-scope (proposed):

- Billing and payment surfaces — third-party processor integration,
  not ours to test on camera
- Any admin console used by Pensar staff — its existence may be
  acknowledged but it is not a target
- Any internal infrastructure dashboard, observability stack, or CI
  control plane
- Any endpoint that proxies to a model provider — provider terms apply
- Any subsystem that, even in staging, holds data from a real customer

Every item above is an assumption. Before this scenario advances, the
Console team writes the actual scope statement, and that statement
overrides anything in this list.

### Stack & Architecture

Characterized at a high level only, since specifics are unverified.
Assumed shape, to be confirmed:

- Web frontend served at a `console.*` hostname
- API at an `api.*` hostname or under an `/api` prefix
- Auth backed by a session store and an HMAC verification layer for
  signed CLI requests
- Webhooks endpoint(s) for agent callbacks
- A relational database with tenant-scoped rows
- Object storage for findings artifacts
- A queue or task runner for asynchronous work
- Standard cloud hosting (provider unspecified in this spec)

The threat model below is written against this shape. If the actual
architecture differs materially, the threat model is rewritten.

### Threat Model

Assets:

- Customer findings data and pentest artifacts in the staging clone
  (synthetic, but representative in shape)
- Authentication material for staging accounts
- HMAC verification keys used by the Console for CLI requests
- Webhook signing secrets
- Workspace and membership records
- Audit log integrity

Trust boundaries:

- Unauthenticated internet vs. Console edge
- Authenticated user vs. another authenticated user (tenant isolation)
- Authenticated user vs. authenticated agent (different actor class)
- Console API vs. internal services (out of scope here)

Adversaries:

1. Unauthenticated attacker on the public internet
2. Authenticated user of a different workspace (cross-tenant)
3. Authenticated user inside a workspace attempting privilege
   escalation within that workspace
4. Compromised CLI client posting crafted HMAC-signed requests
5. Compromised webhook source posting crafted callbacks
6. Subdomain takeover or DNS-related external adversary

Non-adversaries (out of scope for this engagement):

- Insiders with legitimate production access
- Cloud provider compromise
- Anyone with physical access to staging hardware

Assumptions:

- The staging clone is network-isolated from production
- Staging credentials cannot authenticate against production
- No production data is present in staging
- The Console team has acknowledged the engagement window and is on
  call for the duration

### Plausible Findings

As with Scenario A, this is a candidate-bug list, not a claim about
Console. It is anchored in classes of findings that have repeatedly
affected SaaS services with similar shapes. None are asserted to
exist; they describe what the tool would plausibly look for. Same
legend (E / B / I) as Scenario A.

1. (E) IDOR on a workspace-scoped resource with sequential or
   guessable IDs and a loosely-applied workspace authorization check.
   Reference class: ubiquitous SaaS multi-tenant bug.

2. (E) Webhook endpoint accepting unsigned callbacks, or verifying
   signatures with a timing-unsafe comparison, or not binding the
   signature to body length.

3. (E) HMAC verification on Console mirroring the CLI-side weakness
   from Scenario A — partial canonicalization allowing cross-operation
   replay.

4. (E) Device-flow `device_code` endpoint without per-IP rate limiting,
   allowing brute-force of short codes.

5. (E) Session cookie without `Secure`/`HttpOnly`, or with permissive
   `SameSite`.

6. (E) Subdomain takeover on a `*.pensar.dev` or marketing hostname
   pointing at a deprovisioned third-party service.

7. (E) Reflected or stored XSS on a workspace-scoped UI surface
   rendering user-controlled content without contextual escaping.

8. (I) Tenant isolation gap in object storage — signed URLs whose
   prefix leaks cross-tenant identifiers, or with excessive expiry.

9. (I) Race condition in workspace invitation acceptance allowing
   acceptance of a revoked invitation.

10. (I) SSRF via a feature that fetches a user-supplied URL (avatar,
    webhook test, OG image fetch). Endemic in URL-input fields.

11. (I) Information disclosure in error responses including stack
    frames or internal hostnames.

12. (B) Missing security headers on the marketing site (CSP, HSTS).

13. (B) Cookie attribute warnings on non-sensitive cookies.

14. (B) Outdated JS bundle reference surfaced by attack-surface mapping
    but no longer reachable from the app.

15. (I) CSRF on a state-changing JSON endpoint that accepts
    `application/json` cross-origin without a custom-header
    requirement.

The on-camera narrative for this scenario depends heavily on what we
are allowed to find and show. The realistic editorial outcome is one
or two (B) findings shown, one (I) finding discussed at high level, and
the (E) findings — if any — handled entirely off camera through the
internal vulnerability process, with the episode acknowledging that
they exist and were fixed without disclosing details until the standard
disclosure window has passed.

### Recording Strategy

Pre-recording: staging clone is provisioned and seeded with synthetic
fixtures (users, workspaces, memberships, findings, webhooks) with no
data derived from real customers; recording window is scheduled with
the Console team on call; production paging is muted for the recording
host; DNS for staging hostnames is verified to resolve only inside
staging, and the recording host has no credentials that would
authenticate against production.

During recording: Apex is invoked with the staging Console endpoint
configured explicitly, and the hostname is shown on camera so the
audience knows this is staging; the TUI shows agent reasoning and the
findings registry; network panels (if shown) are pre-filtered for
tokens, cookies, and signed URLs; Playwright frames are reviewed live
by a designated reviewer with a "hold" kill switch; findings registry
shows titles and CVSS but masks reproduction steps and bodies until
edit.

Real-time secret redaction: cookies, tokens, signed URLs, and HMAC
signatures are masked in the recording stream before durable storage,
including inside agent reasoning text; internal-pattern hostnames may
be re-aliased in post.

Handling a real critical finding mid-recording: same "hold" protocol
as Scenario A. Additionally, the Console team is notified within the
engagement window to begin remediation in parallel. If the finding
implicates production by analogy (same code path), the engagement
pauses for an emergency remediation cycle, separately tracked.

Cut in post: all raw HTTP traffic frames; references to internal
staff, channels, or tooling; frames with real or staging-shaped
cookies or bearer tokens; references to customer names even in
fixtures (even where a synthetic fixture name happens to collide with
a real customer); moments where Apex output discloses an unfixed
vulnerability in detail.

### Cleanup Plan

After the episode wraps: (1) staging clone torn down and synthetic
data destroyed; (2) engagement credentials rotated and revoked; (3)
findings filed in the internal tracker under the standard
vulnerability process with the engagement referenced; (4) high or
critical findings follow standard disclosure timelines, and
audience-disclosed findings are limited to those past the disclosure
window or intentionally seeded; (5) post-mortem covers the engagement
and resulting product changes; (6) Console team and security review
and sign off on the recording before publish; (7) retrospective on the
production process itself, including whether the format is worth
repeating.

## Production checklist for self-referential demos

The following checklist applies to both scenarios. Items marked (A) or
(B) apply to that scenario only; unmarked items apply to both.

Pre-production:

- [ ] Editorial brief written and circulated
- [ ] Engineering leadership sign-off captured in writing
- [ ] Security sign-off captured in writing
- [ ] Legal sign-off captured in writing
- [ ] (B) Console team sign-off captured in writing
- [ ] (B) Staging clone provisioned, isolated, and verified
- [ ] (B) Synthetic fixture set generated and reviewed for accidental
      real-customer collisions
- [ ] (A) `apex-target-vulnerable` branch updated; seeded-bug list
      reviewed and stored privately
- [ ] Threat model document finalized and committed to the demo asset
      set
- [ ] Recording machine provisioned as single-purpose, with no
      personal credentials present
- [ ] Synthetic `~/.pensar/` directory prepared
- [ ] Mock or staging endpoint wiring verified — no path to production
- [ ] Real-time redaction tooling configured and tested with a dry run
- [ ] Reviewer with kill switch identified and briefed
- [ ] Pass-one (off-camera) run scheduled (A only)
- [ ] Engagement window scheduled and announced internally (B only)
- [ ] Production paging muted for the recording host (B only)

Recording day:

- [ ] Verbal "hold" protocol reviewed with all participants
- [ ] Recording machine state captured before first take
- [ ] Each take logged with timestamps
- [ ] Reviewer present for every take
- [ ] No personal devices in the recording environment

Post-production:

- [ ] Redaction pass applied to all takes
- [ ] Editorial review pass
- [ ] Security review of the cut
- [ ] Legal review of the cut
- [ ] (B) Console team review of the cut
- [ ] Findings reconciliation: seeded vs. surfaced (A); engagement
      findings filed (B)
- [ ] Patching agent diffs reviewed and either landed or discarded
- [ ] Recording machine wiped
- [ ] Staging clone torn down (B)
- [ ] Credentials rotated (B)
- [ ] Post-mortem written
- [ ] Publish-time gate: no unfixed undisclosed criticals tied to the
      episode

Publish:

- [ ] Episode description includes the dogfooding framing and the
      scope statement
- [ ] (A) Episode discloses the existence of `apex-target-vulnerable`
      and explains why
- [ ] (B) Episode discloses that the run was against a staging clone
      with synthetic data
- [ ] Links to the post-mortem and to any landed fixes are included in
      the description
- [ ] Comments are monitored for the first 72 hours for any viewer
      report of incidental disclosure

## Open questions

These must be resolved before filming. Listed without prescribed
answers because the answers materially change the spec.

1. Named approvers for Scenario A on engineering and security sides.
   Is one sufficient or are both required for go?

2. Named approver for Scenario B inside the Console team, and a
   separate approver for the staging environment itself.

3. Does legal review threat model documents before they appear on
   camera, or only the final cut?

4. Disclosure timeline policy for pass-one findings against `main`.
   Is a finding surfaced by our own tool against our own code on the
   same clock as an externally reported bug, or a different clock?

5. Does a Console staging clone exist today, or does it need to be
   built? If built, who owns the work and what is the timeline?

6. For Scenario B, is there a precedent for running adversarial tooling
   against staging during business hours, or is out-of-hours required?

7. Owner of the `apex-target-vulnerable` branch — security, Apex
   engineering, or demo production? Ownership affects who introduces
   seeded bugs and who reviews them.

8. Policy if pass-one surfaces a finding we cannot fix before the
   intended publish date. Does the episode slip or ship without
   disclosing the finding until the fix lands?

9. For Scenario B, policy if Apex exhibits a behavior during the
   engagement that is itself a bug in Apex (not Console) — do we treat
   it as a Scenario A finding mid-Scenario B?

10. Specific list of internal hostnames, codenames, and staff names
    that must never appear on camera, and the maintainer of that list.

11. Do we publish the threat models from these episodes as standalone
    artifacts on docs.pensar.dev, or keep them as on-camera-only?

12. For the Scenario A patching-agent diff: must the diff actually
    land in `main` (where applicable) before publish, or is showing
    the candidate diff sufficient?

13. Finale completion criterion: both episodes shipping, or is
    Scenario A alone acceptable if Scenario B is not approved?

14. Who handles audience comments after publish — designated responder
    for the first week, and the line between "answer in comments" and
    "redirect to security contact."

15. Are these episodes one-shot finales or a recurring cadence
    (annual, per major release)? Affects investment in reusable
    infrastructure (staging clone tooling, vulnerable-fork tooling,
    redaction pipelines).
