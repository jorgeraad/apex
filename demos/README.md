# Apex Demo Series — Plan and Specs

This directory holds the planning and per-app design specs for the Apex demo
video series. Each spec under `specs/` describes one realistic-looking demo
target: its premise, stack, architecture, data model, intentional
vulnerabilities, and the Apex features the demo is designed to showcase.

Specs are written for the engineer who will scaffold the target before
filming. They are not user-facing copy.

---

## Why we are doing this

Apex is the most capable AI-powered pentest CLI we ship, and most of its
strongest features only become visible when it is pointed at something that
looks like a real application. Static screenshots and feature-by-feature
docs do not communicate the value of:

- a sub-agent swarm exploring an attack surface in parallel,
- the findings registry merging evidence from multiple agents,
- the patching agent closing the loop on a fix,
- the judge agent suppressing false positives,
- whitebox-vs-blackbox reasoning on the same target,
- `/operator` chaining exploits in real time,
- the headless CLI gating a real PR in CI.

A recurring demo series is the cheapest way to make all of that legible — at
once to developers, security engineers, AI builders, and prospective Pensar
Console customers. The target audience does not want yet another "vulnerable
app for training" walkthrough; they want to see Apex work on apps that look
like the ones they ship.

The series also doubles as a content factory. Every demo produces:

- a long-form video (3-12 min depending on format),
- a 60-second social cut,
- one or more screenshots for the README and docs,
- a written report artifact (Apex's own output) that can be embedded in docs,
- raw footage that sales can clip into decks.

By keeping the targets industry-realistic and the bugs tied to real-world
CVEs and bounty reports, every demo also accumulates credibility: viewers
see Apex find the same shape of bug they have seen in actual postmortems.

---

## Goals

In priority order:

1. **Prove Apex works on real-shaped apps.** Targets must look like SaaS, not
   training dojos. Bug classes must mirror real-world CVEs and bounty reports.
2. **Show breadth.** Across the series, cover ~14 distinct backend ecosystems
   and 10+ frontend technologies. No two consecutive episodes should feel
   like reruns of the same stack.
3. **Show restraint.** Demos must include scope-guard, finding-judge, and
   "honest target" episodes that show Apex declining to fabricate findings.
   Trust is the long-term moat.
4. **Be reusable.** Specs are designed so that one scaffold can support
   multiple episodes (whitebox vs blackbox, threat-model variant, patching
   variant, CI variant).
5. **Be funny sometimes.** The series carries better with personality. About a
   third of the episodes use lighthearted premises (CatBnB, Plantr,
   ChronoMart) — but the bugs underneath are still real.

---

## Audience

- **Primary:** developers shipping web apps; application-security engineers.
- **Secondary:** AppSec leads, eng managers, AI builders, devtools buyers.
- **Tertiary:** bug-bounty hunters, security researchers, conference
  organizers looking for talk material.

Each spec calls out which audience the episode is tilted toward.

---

## Series structure

- **Cadence:** weekly, with mixed formats so a single recording day can yield
  one long-form and two short-form episodes.
- **Format mix:**
  - 60-second cold opens (install → first finding) — highest reach.
  - 3-6 min feature spotlights (one slash command, one tool, one agent).
  - 8-12 min end-to-end runs (full engagement on a target).
  - 15-25 min "director's cut" (post-mortem, design rationale, behind the
    scenes — for the channel's deeper audience).
- **Categories** (each spec is tagged):
  - Industry verticals (fintech, healthcare, edtech, e-commerce, devtools,
    HR, gov, b2b chat, PM, AI SaaS).
  - Funny/quirky (CatBnB, Plantr, SpectraView, DeepEats, Plundr, ChronoMart,
    SnailStrider, UnboundShelf).
  - Hostile / edge-case (WAF-fronted, pure SPA, SSO+MFA, rate-limited,
    hardened Rust, planted herrings).
  - Self-referential (Apex pentests Apex; Apex pentests Pensar Console).

---

## Format conventions

These conventions apply across every demo unless a spec calls out an exception.

- **Cold open ≤ 8 seconds.** State the target's industry and the headline
  question in one breath. Example: "fintech app, Spring Boot, end users can
  see other accounts' transactions — let's see if Apex finds it."
- **Tone matches the target.** Serious tone for industry verticals; playful
  tone for funny ones; reflective tone for self-referential. Hostile/edge-case
  episodes use the tone of the "host" target.
- **Theme.** Default to the dark theme. Switch to light only if recording
  conditions require it. Do not change theme mid-episode.
- **Terminal size.** Standard recording is 120×40. The "tiny terminal"
  episode (separately tracked) uses 60×20 to demonstrate resilience.
- **Latency.** Always record on a real model run, not a replay. Cuts allowed,
  but never simulate output. The series' credibility depends on this.
- **Findings reporting.** End every full-engagement episode by opening the
  generated report. The artifact is the payoff.
- **No emojis on terminal.** Do not log emojis to stdout in any scaffolded
  demo target. Emojis in episode titles are fine.
- **Closing card.** Always end with: target type, time-to-first-finding,
  total findings by severity, and a 2-second pause on the report.

---

## Ecosystem coverage matrix

The series intentionally rotates ecosystems. By episode 18 of the main
sequence, viewers have seen Apex meaningfully engaged with:

| Backend                 | Demo                          | Frontend             |
| ----------------------- | ----------------------------- | -------------------- |
| Kotlin / Spring Boot    | VaultLine (fintech)           | React + TS           |
| Java / Spring Boot      | MedVault (healthcare)         | Angular              |
| Ruby on Rails / Hotwire | Atlas LMS (edtech)            | Hotwire              |
| PHP / Laravel + Livewire| ShopHearth (e-commerce)       | Livewire             |
| Go / Connect+gRPC       | PipelineIQ (CI/CD)            | React + TS           |
| C# / .NET 8 + Blazor    | PeoplePort (HR)               | Blazor Server        |
| Python / Django + HTMX  | BenefitBridge (gov)           | HTMX                 |
| Elixir / Phoenix LiveView | RelayChat (b2b chat)        | LiveView             |
| Node / NestJS + GraphQL | ThreadOps (PM)                | React + TS           |
| Python / FastAPI        | PromptForge (AI SaaS)         | Next.js App Router   |
| Next.js + Supabase      | CatBnB (funny)                | Next.js              |
| Node / Express + Mongo  | Plantr (funny)                | React Native         |
| Python / Flask          | SpectraView (funny)           | jQuery               |
| AWS Lambda + DynamoDB   | DeepEats (funny)              | Vue 3                |
| Scala / Play            | Plundr (funny)                | scalajs-react        |
| WordPress / WooCommerce | ChronoMart (funny)            | WP theme             |
| Kotlin / Ktor           | SnailStrider (funny)          | Flutter + Vue        |
| SvelteKit / PocketBase  | UnboundShelf (funny)          | SvelteKit            |
| Rust / Axum             | Edge-case: hardened target    | minimal SSR          |

Plus: Vue 3 SPA (edge-case), Spring Boot + Okta SAML+WebAuthn (edge-case),
Cloudflare WAF in front of Laravel (edge-case), TypeScript + Bun + React Ink
(self-referential — the actual Apex CLI), and whatever Pensar Console runs.

---

## Index of demo specs

Industry verticals (the "headline" episodes):

| # | Demo            | Industry        | Stack (short)                    | File                                    |
| - | --------------- | --------------- | -------------------------------- | --------------------------------------- |
| 1 | VaultLine       | Fintech         | Kotlin · Spring Boot · React     | [specs/01-vaultline-fintech.md](specs/01-vaultline-fintech.md) |
| 2 | MedVault        | Healthcare      | Java · Spring · Angular · FHIR   | [specs/02-medvault-healthcare.md](specs/02-medvault-healthcare.md) |
| 3 | Atlas LMS       | EdTech          | Rails · Hotwire                  | [specs/03-atlas-lms.md](specs/03-atlas-lms.md) |
| 4 | ShopHearth      | E-commerce      | Laravel · Livewire · MySQL       | [specs/04-shophearth-ecommerce.md](specs/04-shophearth-ecommerce.md) |
| 5 | PipelineIQ      | DevTools CI/CD  | Go · Connect+gRPC · K8s          | [specs/05-pipelineiq-cicd.md](specs/05-pipelineiq-cicd.md) |
| 6 | PeoplePort      | HR / Payroll    | .NET 8 · Blazor · SQL Server     | [specs/06-peopleport-hr.md](specs/06-peopleport-hr.md) |
| 7 | BenefitBridge   | Government      | Django · HTMX · Login.gov-style  | [specs/07-benefitbridge-gov.md](specs/07-benefitbridge-gov.md) |
| 8 | RelayChat       | B2B Chat        | Phoenix · LiveView               | [specs/08-relaychat-b2bchat.md](specs/08-relaychat-b2bchat.md) |
| 9 | ThreadOps       | Project Mgmt    | NestJS · GraphQL · Prisma        | [specs/09-threadops-pm.md](specs/09-threadops-pm.md) |
|10 | PromptForge     | AI SaaS         | FastAPI · Next.js · MCP          | [specs/10-promptforge-ai.md](specs/10-promptforge-ai.md) |

Funny / quirky:

| #  | Demo          | Premise                             | Stack (short)                   | File                                    |
| -- | ------------- | ----------------------------------- | ------------------------------- | --------------------------------------- |
| 11 | CatBnB        | Airbnb but for cats                 | Next.js · Supabase              | [specs/11-catbnb-funny.md](specs/11-catbnb-funny.md) |
| 12 | Plantr        | Tinder for houseplants              | Express · Mongo · React Native  | [specs/12-plantr-funny.md](specs/12-plantr-funny.md) |
| 13 | SpectraView   | Yelp for haunted houses             | Flask · SQLite                  | [specs/13-spectraview-funny.md](specs/13-spectraview-funny.md) |
| 14 | DeepEats      | DoorDash for conspiracy theories    | AWS Lambda · DynamoDB           | [specs/14-deepeats-funny.md](specs/14-deepeats-funny.md) |
| 15 | Plundr        | LinkedIn for pirates                | Scala · Play · Akka             | [specs/15-plundr-funny.md](specs/15-plundr-funny.md) |
| 16 | ChronoMart    | Etsy for time travelers             | WordPress · WooCommerce         | [specs/16-chronomart-funny.md](specs/16-chronomart-funny.md) |
| 17 | SnailStrider  | Strava for procrastinators          | Kotlin · Ktor · Flutter         | [specs/17-snailstrider-funny.md](specs/17-snailstrider-funny.md) |
| 18 | UnboundShelf  | Goodreads for banned books          | SvelteKit · PocketBase          | [specs/18-unboundshelf-funny.md](specs/18-unboundshelf-funny.md) |

Combined / specialized:

| #  | Demo            | Purpose                              | File                                    |
| -- | --------------- | ------------------------------------ | --------------------------------------- |
| 19 | Edge cases      | 6 hostile/edge-case scenarios        | [specs/19-edge-cases.md](specs/19-edge-cases.md) |
| 20 | Self-referential| Apex on Apex; Apex on Console        | [specs/20-self-referential.md](specs/20-self-referential.md) |

---

## Suggested running order (12 episodes)

A starter season that establishes credibility, then expands into variety.
Episode numbers refer to recording/release order, not spec numbers.

1.  **The 60-second pitch** — install → first finding on a Juice Shop instance.
    Quick promo cold-open before the season formally starts.
2.  **VaultLine** (spec 1) — `/pentest` end-to-end on a fintech app. The
    headline episode.
3.  **Whitebox vs blackbox** — same target (Atlas LMS, spec 3), both modes
    side-by-side. Use the same scaffold; record twice.
4.  **`/operator` deep-dive** — manual exploitation chain on ShopHearth
    (spec 4). Coupon stacking + webhook spoof + IDOR chained.
5.  **Swarm mode** — visualize parallel sub-agents lighting up against
    PipelineIQ (spec 5).
6.  **CatBnB** (spec 11) — Supabase RLS misconfiguration in 60 seconds.
    First "funny" episode; lighter tone, same engineering rigor.
7.  **The patching agent** — find → fix → re-test on Plantr (spec 12).
8.  **CI gate** — copyable GitHub Actions workflow that fails a PR because
    Apex caught an IDOR. Headless CLI episode.
9.  **Auth maze** — SSO + MFA target from edge-cases (spec 19, scenario 3).
    `authenticationAgent` walkthrough.
10. **The honest target** — hardened Rust + Axum from edge-cases (spec 19,
    scenario 5). Apex declines to fabricate findings.
11. **PromptForge** (spec 10) — prompt injection + MCP tool abuse. Pulls
    in the AI-builder audience.
12. **Apex pentests Apex** (spec 20, part 1) — meta finale for the season.

Subsequent seasons rotate among MedVault, BenefitBridge, RelayChat,
ThreadOps, PeoplePort, plus the remaining funny ones.

---

## Production checklist (per demo)

Pre-record:

- [ ] Spec reviewed. Bugs in the scaffold match the spec.
- [ ] All intentional vulns confirmed exploitable manually first.
- [ ] Apex run dry on staging copy. Record the time-to-first-finding.
- [ ] Scope-guard verified: target domain is in-scope, nothing outside is.
- [ ] Reset script in place (DB seed + container restart).

Record:

- [ ] Cold open recorded last (after main run, when delivery is sharp).
- [ ] Terminal size locked at 120×40.
- [ ] No personal API keys visible in the env.
- [ ] Apex version pinned; note it in the closing card.
- [ ] If the run uses `--threat-model`, file is in the repo.

Post-record:

- [ ] 60-second cut produced.
- [ ] Report artifact extracted, redacted if needed, embedded in docs.
- [ ] Title, time-to-first-finding, finding count logged in the series log.

---

## How to add a new demo

1. Decide the demo's category (industry / funny / edge-case / self-ref) and
   its differentiator within the series.
2. Pick a stack from the matrix above. Prefer ecosystems not yet covered.
3. Draft the spec following the same template the existing specs use:
   premise → stack → architecture → data model → routes → auth →
   intentional vulnerabilities → real-world parallels → Apex features →
   storyline → build notes → recording notes.
4. Have at least one engineer and one security reviewer sign off on the
   intentional-vulns list before the scaffold is built.
5. Add the spec to the index above and to the running order.

---

## Open questions

- **Hosting.** Should the scaffolds live in this repo (`demos/scaffolds/`)
  or in separate per-app repos under the org? Lean: separate repos so
  vulnerable code never accidentally ships in a release tarball of the
  Apex CLI.
- **Reset story.** A single Docker compose per scaffold is the minimum.
  Do we want a shared "demo harness" CLI that can `seed`, `reset`, and
  `verify-bugs` against any scaffold?
- **Self-referential safety.** Apex-pentests-Pensar-Console requires
  written approval and probably a staging clone. Block on that before
  filming.
- **Pacing.** How often is "weekly" actually sustainable? Confirm with
  whoever owns production capacity before we commit publicly.

These are tracked in `specs/20-self-referential.md` and the per-spec
"Build Notes" sections where they apply.
