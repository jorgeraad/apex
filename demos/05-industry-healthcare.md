# Demo 05 — Industry audit: pentesting a healthcare API

**Length:** 6–7 minutes
**Format:** Narrated screencast w/ document inserts
**Audience:** Security engineers, CISOs, compliance-adjacent roles
**Series:** **Industry Audit** — episode 1 of an ongoing format

---

## Purpose

Make Apex feel like a real consulting deliverable — the kind a sec-eng team
would hand to compliance. Frame it as a "fictitious engagement against a
healthcare app" so viewers in regulated industries see themselves in the
story.

This episode also establishes the Industry Audit format. Subsequent episodes
swap the vertical: financial, retail, government, AI startup, etc.

---

## Requirements

- **Asset:** `vuln-health` — FastAPI healthcare records API. Planted bugs:
  - **IDOR** on `GET /patients/{id}/records` (any auth'd user can read any
    patient).
  - **Auth bypass** on `/admin/audit` (forgotten dev endpoint, no auth check).
  - **Weak password policy** in `/auth/register` accepts `password`.
  - One *false positive* trap: an endpoint that *looks* vulnerable but isn't.
    The agent should correctly *not* report it.
- **Pre-written threat model** at `./vuln-health-threat-model.md`. Frames the
  app as PHI-handling, FHIR-ish, with HIPAA-relevant trust boundaries.
- **Scope file** with strict allowlist:
  ```
  hosts: api.medrecord-demo.local
  ports: 443
  strict: true
  ```
- **Custom report theme** that styles the final report with a "MedRecord Inc."
  letterhead and HIPAA-flavored language in the executive summary section.
- **Insert graphics:** an opening "engagement scope" doc and a closing
  "executive summary" page.

---

## Things to check first

- [ ] All three planted bugs reproduce manually with `curl`.
- [ ] The false-positive trap is configured — confirm a recent dry run did
      *not* report it. If the agent flags it, dial down the trap or pick a
      different one. False-positive resilience is part of the story we're
      telling.
- [ ] `vuln-health` resets cleanly. Patient seed data should *look* realistic
      — names, DOBs, MRNs — but be obviously fake on inspection (Donald Duck,
      DOB 1934-06-09, etc.).
- [ ] `/obfuscate on` is set before the first frame of footage. PHI-shaped
      strings must redact, even though they're fake. We are training viewers
      to associate Apex with safe demoing.
- [ ] Pre-written threat model loads with `--threat-model @./...` and
      noticeably shapes the agent's plan (tasks reference threat-model
      sections by name).
- [ ] Custom report theme renders cleanly in the report viewer dialog
      *and* in the saved Markdown file.
- [ ] Strict scope mode actually blocks an out-of-scope request — record a
      brief out-of-scope attempt for Scene 5.

---

## Full script

### Scene 1 — Industry Audit stinger (0:00–0:08)

**Visual:** Industry Audit intro card:

```
INDUSTRY AUDIT · EPISODE 1
HEALTHCARE
```

**VO:** "Industry Audit. Episode one. Today we're pentesting a fictitious
healthcare records API."

---

### Scene 2 — Engagement framing (0:08–0:50)

**Visual:** A document slides into frame — "MedRecord Inc. Engagement Scope."
Bullet points:

```
- Target: api.medrecord-demo.local
- Surface: PHI access, auth, admin
- Scope: 443/tcp only, strict allowlist
- Threat model: pre-written (see attached)
- Constraint: read-only — no destructive payloads
```

**VO:** "Pretend you're a security engineer at MedRecord Inc. The app handles
patient records. The threat model is already written. The scope is locked
down. You have an afternoon."

Cut to terminal.

```bash
ls
# vuln-health-threat-model.md  scope.txt
```

---

### Scene 3 — Launch with constraints (0:50–1:30)

```bash
pensar
```

In TUI:

```
/pentest --target https://api.medrecord-demo.local \
         --threat-model @./vuln-health-threat-model.md \
         --strict \
         --hosts api.medrecord-demo.local \
         --ports 443
```

**VO:** "We're handing the agent the threat model, the host allowlist, and
strict mode. Strict mode means anything outside scope gets blocked at the
tool layer — the agent can't accidentally test a third party's infrastructure
even if it tries."

Quick zoom on the flags row at the bottom of the screen showing all the
applied constraints.

---

### Scene 4 — Plan tied to the threat model (1:30–2:30)

Agent's plan view appears. Tasks are shaped by the threat model — note that
they reference threat-model sections by name:

```
TASK 1 · TM§3.1 PHI access controls
TASK 2 · TM§4.2 Admin surface
TASK 3 · TM§2.4 Auth & session
```

**VO:** "Look at this — the tasks are anchored to threat-model sections.
That's not magic; the agent reads the threat model the way an analyst would.
It plans against your concerns, not its own."

Show the swarm spinning up. Sub-agent tiles populate.

---

### Scene 5 — Strict scope demo (2:30–3:00)

Cut to a moment where the agent attempts to follow a redirect to a
third-party CDN host. Scope guard fires — visible toast:

> ⚠ Out of scope: cdn.healthcare-marketing.com (blocked)

**VO:** "Watch this. The app redirected to a marketing CDN. Strict mode
blocked it. In a real engagement, this is the line between 'pentest' and
'unauthorized access of a third party.' Apex won't cross it."

---

### Scene 6 — IDOR finding (3:00–3:50)

Time-lapse swarm activity. Then real-time when a finding lands:

> **🔴 Critical · Insecure Direct Object Reference**
> `GET /patients/{id}/records`
> Authenticated user `alice` retrieved records for patient `donald-duck`.
> No ownership check.

PHI-shaped strings in the evidence are redacted by `/obfuscate`.

**VO:** "First finding. Insecure direct object reference. Any authenticated
user can pull any patient's records. In the real world, this is the kind of
bug that ends up in a breach disclosure."

---

### Scene 7 — Auth bypass on admin (3:50–4:30)

Second finding:

> **🔴 Critical · Authentication Bypass**
> `/admin/audit` is reachable without authentication.
> Likely a forgotten dev route.

**VO:** "Second finding — and this is a great one to call out because it's so
common. A `/admin/audit` route, no auth, probably committed by someone three
years ago and nobody noticed. Apex finds it because it doesn't *assume*
anything about the auth boundary — it tests."

---

### Scene 8 — Weak password (4:30–5:00)

Third finding:

> **🟡 Medium · Weak Password Policy**
> `/auth/register` accepts `password`, `12345678`, `qwertyui`.
> No minimum complexity enforced.

**VO:** "Third finding — weak password policy. Lower severity, easy fix,
still worth filing."

---

### Scene 9 — The non-finding (5:00–5:30)

**Visual:** Scroll back through the agent log to a moment where it tested the
false-positive trap (e.g., a `?debug=true` query that *looks* dangerous but
hits an empty handler). The agent's notes:

```
Tested ?debug=true — returns 200 OK with empty body.
Looked promising. Confirmed not exploitable. Not reporting.
```

**VO:** "Worth showing what *didn't* happen. There was a tempting query
parameter that looked like a debug toggle. Apex tested it, confirmed it
wasn't exploitable, and didn't file a finding. False-positive discipline is
half of why a tool is worth using."

---

### Scene 10 — The deliverable (5:30–6:30)

Cut to the report. Custom theme — MedRecord letterhead at top, executive
summary, then per-finding detail. Scroll through. Linger on the executive
summary which uses HIPAA-flavored language ("PHI exposure," "minimum
necessary," "audit log integrity").

**VO:** "And here's the deliverable. Executive summary up front, framed in
the language your compliance team uses. Per-finding evidence, severity, and
suggested fix below. This is what you hand to the engineering lead and the
CISO. Same artifact, two audiences."

---

### Scene 11 — Outro (6:30–6:50)

**Visual:** Industry Audit outro card.

```
NEXT EPISODE
RETAIL E-COMMERCE
```

**VO:** "Industry Audit, episode one. Next time, we go retail."

---

## Editing notes

- Lean into the consulting-deliverable framing. Document inserts (scope page,
  exec summary) sell that more than slick visual effects.
- Scene 9 — the non-finding — is the most important scene for sec-eng
  viewers. Don't cut it, even if the demo runs long.
- Keep `/obfuscate on` framing visible somewhere subtle — a small icon in
  the TUI status bar — so viewers learn the feature exists.
- Pre-record the engagement-scope doc and exec-summary doc as separate
  graphics; don't try to capture them live in VS Code.
- Leave the "Pensar Console" plug for the deliverable scene's outro: "If
  you're running this against a real surface, Console gives you scheduling
  and history. Link below."
