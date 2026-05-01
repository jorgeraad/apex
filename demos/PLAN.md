# Apex Demo Series — Plan & Framework

A recurring series of demos showcasing Pensar Apex. Each demo is scoped, scripted,
and rooted in a reusable asset library so episodes can ship on a regular cadence
without rebuilding the world each time.

---

## Goals

1. **Build muscle memory.** Every episode reinforces "open terminal → run `pensar`."
2. **Show range.** A library that spans single-feature spotlights, full workflows,
   industry-flavored case studies, internals deep-dives, and lighter/funny clips.
3. **Be reproducible.** Anything we show on screen should be backed by a runnable
   asset (vulnerable demo app, benchmark, sample CI repo, etc.).
4. **Make Apex feel like a thinking partner**, not a button. Show reasoning,
   approval gates, evidence quality — not just final findings.

---

## What Apex is (the substrate every demo draws from)

AI-powered penetration testing — autonomous agents that run blackbox or whitebox
pentests directly from the terminal. TUI on top, headless CLI underneath, Pensar
Console for the cloud edition.

### Audiences (pick one per demo, don't try to hit all three)

- **Developers shifting left** — wants `/pentest` to be as routine as `npm test`.
- **Security engineers** — wants automation across large attack surfaces.
- **Professional pentesters** — wants `/operator` mode for steerable, expert-driven
  testing.

### Headline capabilities a demo can spotlight

- `/pentest` — autonomous swarm (discovery → swarm → report)
- `/operator` — interactive single-agent mode with approval gates
- `/threat-model` — application-centric whitebox threat modeling
- Sub-agent swarm with bounded concurrency + live findings registry
- 7-step methodology (PLAN → VERIFY → PREPARE → TEST → EXPLOIT → DOCUMENT → FINISH)
- Multi-provider model support (Anthropic / OpenAI / Bedrock / vLLM)
- Kali Linux container with preconfigured tools
- 18 themes, light/dark mode, obfuscation/redaction mode
- Approval tiers, strict scope mode, custom headers
- Memory system, sessions, resume
- W&B Weave tracing
- Argus benchmark suite (60 targets, public scores)
- Headless CLI for CI; Pensar Console handoff for cloud

---

## Framework — six dimensions to vary across the series

Pick a value on each axis when scoping a new demo. Same axes = recognizable
series; mix them up = variety.

| Dimension    | Options                                                                                                 |
| ------------ | ------------------------------------------------------------------------------------------------------- |
| **Audience** | dev / sec-eng / pentester / CISO / general technical viewer                                             |
| **Format**   | screencast w/ VO · talking-head w/ inset terminal · split-screen · "raw run" timelapse · live-streamed  |
| **Length**   | 30–60s (social) · 2–3 min (feature) · 5–8 min (workflow) · 15+ min (deep dive)                          |
| **Surface**  | TUI · CLI · Console handoff · container · CI integration · architecture/internals                       |
| **Tone**     | authoritative · playful · suspenseful · "fly-on-the-wall" · educational                                 |
| **Target**   | purpose-built demo app · Argus benchmark · OSS vulnerable lab · staged "industry" clone                 |

### Production rules of thumb

- Always run `/obfuscate on` for any demo using a real-feeling target — protects
  against accidentally leaking demo artifacts in screenshots.
- Consistent intro card (5s logo, `apex` theme) makes the series feel like a series.
- Show the `pensar` command being typed every episode. Repetition builds the brand.
- Show the **finding** before the **exploit chain**. Punchline first.
- End with a single CTA — install line *or* Console link, never both.
- Default theme: `apex`, dark mode. Switch to other themes only when the episode is
  about themes (Demo 16).

---

## Reusable asset library

Build these *first*. They pay back across the whole series. An AI agent can
scaffold each one — see `demos/<NN>-*.md` for which assets each demo depends on.

| Asset                     | Purpose                                                              | Used by demos       |
| ------------------------- | -------------------------------------------------------------------- | ------------------- |
| **`vuln-shop`**           | Node/Express + Postgres e-com app, planted SQLi/IDOR/auth/JWT issues | 1, 4, 7, 11, 16     |
| **`vuln-bank`**           | Flask banking app, BOLA + race-condition transfer + hardcoded secrets | 5, 14              |
| **`vuln-health`**         | FastAPI healthcare records API, PHI exposure, IDOR, weak auth        | 5, 17               |
| **`vuln-llm`**            | AI chatbot wrapper with prompt injection, SSRF via tool, key leakage | 9, 18               |
| **`vuln-admin`**          | Internal admin panel, SSO bypass, audit log forgery                  | 12                  |
| **Argus subset**          | 5 hand-picked benchmarks that finish quickly                         | 2, 8, 19            |
| **CI repo**               | Sample GitHub repo with `pentest.yml` workflow that runs on PR       | 6, 13               |
| **Threat-model target**   | Small open-source-style app with clear architecture                  | 3, 15               |
| **Demo overlay package**  | Lower-third name plates, command ribbon, finding pop-card SVG/PNG    | All                 |
| **"Findings explained" deck** | One-slide-per-vuln cards (SQLi, SSRF, IDOR, etc.)                | 4, 7, 11, 18        |

### Asset build priority

1. `vuln-shop` (powers 6 demos)
2. Demo overlay package (every demo uses it)
3. `vuln-llm` (topical, powers 2 demos, no good public alternative)
4. CI repo + preview-deploy stub
5. Argus rerun script that produces deterministic, replayable runs
6. Remaining vulnerable apps and threat-model target

---

## Demo index

| # | Title                                                       | Length     | Series                | Primary asset    |
| - | ----------------------------------------------------------- | ---------- | --------------------- | ---------------- |
| 1 | [Cold start to first finding in 90 seconds](./01-cold-start.md) | 90s    | Standalone            | `vuln-shop`      |
| 2 | [Apex vs Argus: scoring 34/60 benchmarks](./02-argus-vs-apex.md) | 4–5m   | Standalone            | Argus subset     |
| 3 | [`/threat-model` from clone to playbook in 3 min](./03-threat-model.md) | 3m | Standalone        | Threat-model target |
| 4 | [Find the Bug #1: SSRF in an internal SaaS](./04-find-the-bug-ssrf.md) | 5m | Find the Bug      | `vuln-shop`      |
| 5 | [Industry audit: pentesting a healthcare API](./05-industry-healthcare.md) | 6–7m | Industry Audit | `vuln-health`   |
| 6 | [PR-time pentest: CI just caught an IDOR](./06-pr-time-pentest.md) | 4m   | Standalone            | CI repo          |
| 7 | [`/operator` deep-dive: XSS → cookie → admin](./07-operator-chain.md) | 8–10m | Operator Deep-Dive | `vuln-shop`     |
| 8 | [Speedrun: every Argus single-vuln benchmark](./08-speedrun.md) | 60–90s | Speedrun              | Argus subset     |
| 9 | [Pentesting an AI startup: prompt injection in the wild](./09-ai-startup.md) | 5m | Find the Bug | `vuln-llm`     |
| 10 | [Inside Apex #1: why a swarm beats a mega-agent](./10-inside-apex-swarm.md) | 7m | Inside Apex      | Architecture animation |
| 11 | [Inside Apex #2: the 7-step pentest state machine](./11-inside-apex-state-machine.md) | 5m | Inside Apex | `vuln-shop` |
| 12 | [Approval tiers, scope guards, strict mode](./12-trust-and-safety.md) | 4m | Standalone            | `vuln-admin`     |
| 13 | [Pensar in the pipeline: gating production deploys](./13-pipeline.md) | 6m | Standalone           | CI repo          |
| 14 | [Banking app, race conditions, $0 to $10M](./14-bank-race-condition.md) | 5m | Find the Bug    | `vuln-bank`      |
| 15 | [Whitebox + blackbox = 1+1=3](./15-whitebox-vs-blackbox.md) | 4m | Standalone            | Threat-model target |
| 16 | [Customizing Apex: themes, keybindings, your style](./16-themes-and-style.md) | 90s | Standalone | Theme cycle recording |
| 17 | [Pentesting a hospital admin portal that should not exist](./17-hospital-portal.md) | 4m | Industry Audit | `vuln-health` |
| 18 | [Ten prompt injections, ranked](./18-prompt-injections-ranked.md) | 6m | Listicle              | `vuln-llm`       |
| 19 | [Cost of a pentest: tokens, time, and dollars](./19-cost-of-a-pentest.md) | 5m | Inside Apex     | Weave-enabled run |
| 20 | [Q&A with Apex: 10 questions answered live in operator](./20-qa-with-apex.md) | 7–8m | Q&A         | `vuln-shop`      |

---

## Suggested rollout order

Front-load demos that establish credibility and reuse the most assets:

**Phase 1 — establish the brand (week 1–3):** 1 → 16 → 2 → 4
**Phase 2 — feature spread (week 4–6):** 10, 6, 3, 9
**Phase 3 — narrative depth (week 7–10):** 7, 14, 5, 11
**Phase 4 — sprinkles between long-form drops:** 8, 18, 16-style cuts on demand

---

## Per-demo file shape

Each `demos/<NN>-*.md` file has the same four sections so the series stays
consistent and an AI agent can pick up an episode and produce it end to end:

1. **Purpose** — what this episode is *for*. Audience, what they take away.
2. **Requirements** — assets, environment, recordings, overlays, captions.
3. **Things to check first** — pre-flight: are findings displayable, does the
   target reset cleanly, is obfuscation on, is the theme set, etc.
4. **Full script** — scene-by-scene timing, voiceover lines, on-screen text,
   commands typed, cuts.
