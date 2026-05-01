# Demo 20 — Q&A with Apex: 10 questions answered live in the operator

**Length:** 7–8 minutes
**Format:** Q&A screencast w/ host commentary
**Audience:** Pentesters, curious technical viewers, anyone who wants to
*feel* what operator mode is like
**Series:** **Q&A** — episode 1

---

## Purpose

A format experiment. Operator mode isn't just a pentest tool — it's a
chat interface to a security-specialized agent. We treat the agent
like an interview subject and ask it 10 sharp questions about a target.
Apex answers in real time; the host provides one-line commentary on
each answer.

The viewer should walk away thinking *"It's not just an automation
tool. It's a thinking partner."*

This format scales — future episodes can do "10 questions about a
fintech app," "10 questions about a public CVE," etc.

---

## Requirements

- **Asset:** `vuln-shop` (rich enough to support 10 distinct questions)
  or a richer alternative if available. The agent needs to have
  meaningful answers to varied prompts — not all of them about specific
  bugs.
- **Question list** (vetted in a dry run for substance):
  1. What's the riskiest endpoint?
  2. Which response header looks weakest?
  3. If you had five minutes to compromise this, what would you try?
  4. What's the most dangerous *un-tested* assumption I'm making?
  5. What does the auth surface look like?
  6. Where would you place hidden monitoring if you were the defender?
  7. What's the ratio of attack surface that's actually authenticated?
  8. Walk me through what happens when I hit `/checkout`.
  9. What's the cheapest finding for me to fix today?
  10. What would you ask me to clarify about this target?
- **Host commentary card** — small lower-third where you, the host,
  write a one-line take on each answer.
- **Question counter** — top-right, "Question 4 of 10."
- **Theme:** `apex` dark, `/obfuscate on`.

---

## Things to check first

- [ ] Dry-run the full Q&A. Some questions will produce thin answers —
      replace those questions with sharper ones until all 10 yield
      something interesting.
- [ ] The agent has a non-trivial amount of context built up before
      Q1 fires. Either run a quick `/pentest` first and resume in
      operator mode, OR let the agent crawl for 60s before the camera
      starts.
- [ ] All answers are reasonable on a sec-eng's read. If the agent
      hallucinates, you can either edit it out (and say so on screen)
      or leave it in with corrective host commentary — but pick one
      policy and apply it consistently.
- [ ] Host's one-line takes are pre-thought, not improvised. They're
      where the video's voice lives.
- [ ] Total runtime stays under 8 minutes. A long Q&A loses people. If
      a single question's answer runs long, time-lapse the middle.

---

## Full script

### Scene 1 — Q&A stinger (0:00–0:15)

```
Q&A · EPISODE 1
WE ASKED APEX 10 QUESTIONS ABOUT A TARGET
```

**VO:** "Operator mode isn't just a pentest tool. It's a chat with an
agent that has fresh context on your target. Today, we ask it ten
questions. We rate the answers."

---

### Scene 2 — Setup (0:15–0:50)

Cut to terminal. A pentest has just finished. Resume in operator mode.

```
/operator --resume <session>
```

Quick visual: the operator view loads with prior pentest context
(findings sidebar, asset map).

**VO:** "We just ran `/pentest` on this target. Resume in operator
mode — the agent has the full context now: the asset map, the
findings, the auth surface. That's the agent we're going to ask
questions of."

---

### Scenes 3–12 — The questions (0:50–6:30)

Each question gets ~30–35 seconds.

**Format for each:**
1. Question card on screen: `Q1. What's the riskiest endpoint?`
2. Operator types question, presses Enter.
3. Agent streams answer (time-lapse if long).
4. Host's one-line take appears as lower-third overlay.

#### Q1. What's the riskiest endpoint?
- **Agent (excerpted):** "Riskiest is `/api/admin/exec` — it's
  unauthenticated, accepts a `cmd` parameter, and the response leaks
  stderr. I'd start there."
- **Host take:** *Confirms the SQLi finding from earlier. Useful
  triage.*

#### Q2. Which response header looks weakest?
- **Agent:** "`Strict-Transport-Security` is missing entirely.
  `Content-Security-Policy` is `default-src *`. CSP is doing nothing."
- **Host take:** *Honest, but lower priority than functional bugs.
  Worth filing.*

#### Q3. If you had five minutes, what would you try?
- **Agent:** Lists 3 prioritized probes. Reasoning included.
- **Host take:** *This is what `/operator` is for — fast triage with
  a real plan.*

#### Q4. What's the most dangerous un-tested assumption?
- **Agent:** "I haven't been able to confirm whether session cookies
  are scoped per-tenant. If they aren't, that's a multi-tenant
  isolation issue and I'd need authenticated access from a second
  tenant to test."
- **Host take:** *The agent knows what it doesn't know. This is the
  question worth re-asking on every target.*

#### Q5. What does the auth surface look like?
- **Agent:** Walks through the auth flow — login, session, refresh,
  logout. Notes one anomaly.
- **Host take:** *A clean architectural read.*

#### Q6. Where would you place hidden monitoring as defender?
- **Agent:** "Watch `/api/admin/*` for unauth attempts; log full body
  and source IP. Watch failed login bursts on `/auth/login`. Watch
  outbound DNS for any traffic from the app server."
- **Host take:** *Defender-oriented — a useful angle the agent can
  surface naturally.*

#### Q7. Authenticated vs unauthenticated surface?
- **Agent:** "Roughly 40% authenticated, 60% public. The public surface
  is heavier than typical for a SaaS — likely the marketing pages and
  public product catalog."
- **Host take:** *Sets context for prioritization.*

#### Q8. Walk me through `/checkout`.
- **Agent:** Step-by-step, what calls what, what state changes.
- **Host take:** *Useful as developer documentation, even setting
  aside security.*

#### Q9. Cheapest finding to fix today?
- **Agent:** "The missing CSP. One header line. It won't fix anything
  exploitable today but it raises the floor."
- **Host take:** *Pragmatic. Quick wins are real.*

#### Q10. What would you ask me to clarify?
- **Agent:** "Whether the public catalog is supposed to be reachable
  unauthenticated. If yes, fine. If no, that's a finding all on its
  own."
- **Host take:** *Questions back at the operator. Best one of the
  ten.*

---

### Scene 13 — Verdict (6:30–7:20)

Cut to the host (or on-screen card if no host on camera).

**VO:** "Out of ten questions, the agent gave useful answers to all of
them. Two were standout — Q4 and Q10, where it told me what it didn't
know and what to ask back. That's the difference between a tool that
*runs* a pentest and a tool that *thinks alongside* a pentester."

Quick verdict overlay:

```
Useful answers: 10/10
Standout answers: 2 (Q4, Q10)
Hallucinations: 0
```

(If a hallucination did occur, name it honestly.)

---

### Scene 14 — Outro (7:20–7:35)

```
Q&A · EPISODE 1
NEXT: ASK YOUR OWN TARGET
```

**VO:** "Q&A, episode one. Run `/operator` and try this on something
you own. Subscribe — episode two, we go open-source."

---

## Editing notes

- Question cards are the structural element — keep them stylistically
  consistent and brief.
- The host's one-line takes are the personality. Pre-write them in a
  dry run. Improv usually doesn't land in 8 seconds.
- If an agent answer runs long, time-lapse the middle and cut to the
  conclusion. Don't read every word aloud.
- This episode is a strong cross-promote target for the documentation
  — link it from the `/operator` page.
- This format pairs well with Demo 07 (`/operator` deep-dive). If
  scheduling allows, record Demo 07 and Demo 20 in the same operator
  session — same target, same context, no setup tax.
