# Demo 04 — Find the Bug #1: SSRF in an internal SaaS

**Length:** 5 minutes
**Format:** Screencast w/ voiceover, educational interlude
**Audience:** Developers and pentesters
**Series:** **Find the Bug** — episode 1 of an ongoing format

---

## Purpose

Establish the recurring **Find the Bug** format. Same intro card, same outro,
swap the bug each episode. Each episode plants exactly one vulnerability and
asks: *"Can Apex find it?"* The drama is the journey, not the outcome.

This first one teaches SSRF — a class of bug developers reliably underestimate.

---

## Requirements

- **Asset:** `vuln-shop` configured with a planted SSRF on a "fetch external
  product image" endpoint:
  ```
  POST /api/products/preview
  { "imageUrl": "https://example.com/image.jpg" }
  ```
  No URL validation, no allowlist. Reachable internal targets: a fake metadata
  endpoint at `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
  serving plausible-looking dummy credentials.
- **No other planted bugs in the surface area we'll test.** This episode is
  about one bug; clutter would dilute it.
- **Findings explained deck:** "What is SSRF?" card — 30s of animation
  showing how the request leaves the trusted boundary.
- **Find the Bug intro/outro stinger:** 4-second animated logo with title
  variant baked in ("Find the Bug · Episode 1 · SSRF").
- **Theme:** `apex` dark, `/obfuscate on` for any frames showing the metadata
  response.

---

## Things to check first

- [ ] The planted SSRF reproduces reliably with manual `curl` before recording.
- [ ] The mock metadata endpoint returns dummy credentials that *look* real
      (AWS-shaped) but are obviously fake on close inspection. Keep the strings
      short.
- [ ] Apex has actually found this bug at least once on a dry run — agentic
      LLMs aren't deterministic. Expect to record 2–3 takes.
- [ ] The `/operator` mode displays approval gates clearly on screen — this
      episode hinges on the viewer seeing the gate fire.
- [ ] The final report contains a clean SSRF finding with severity, request
      payload, and the metadata response in evidence — that's the hero shot.
- [ ] Confirm the educational SSRF card matches the actual exploit shown — no
      contradictions between the explainer and the live evidence.
- [ ] Run `/obfuscate on` and verify the credential-shaped strings in the
      response get redacted on screen.

---

## Full script

### Scene 1 — Series stinger (0:00–0:05)

**Visual:** Find the Bug intro animation. Title card:

```
FIND THE BUG · EPISODE 1
SSRF
```

**VO:** "Find the Bug. Episode one. We've planted a server-side request
forgery vulnerability. Let's see if Apex can find it."

---

### Scene 2 — The setup (0:05–0:35)

**Visual:** Browser frame showing `vuln-shop`'s admin panel — "Add a new
product. Paste an image URL." A placeholder for the product image renders.

**VO:** "Here's the app. It's an internal SaaS. Sellers paste an image URL
and we render a thumbnail in the product card. Looks innocuous, right?"

Cut to a slide showing the implicated endpoint:

```
POST /api/products/preview
{ "imageUrl": "<any URL>" }

→ server fetches URL server-side, returns image bytes
```

**VO:** "Server-side, the app fetches that URL. No allowlist. No URL
validation. We've planted exactly one bug. Question is whether Apex can spot
it without us telling it where to look."

---

### Scene 3 — Launch operator (0:35–1:05)

**Type:**

```bash
pensar
```

In operator view:

```
/operator --target https://vuln-shop.local
```

**VO:** "We're using `/operator` mode for this one — interactive, single
agent, with approval gates. That way you can watch the agent reason instead
of just delivering a report."

Initial prompt typed:

```
Find authentication issues, IDOR, SSRF, or any input that crosses a trust
boundary. Start with the API surface.
```

---

### Scene 4 — Recon (1:05–1:50)

Agent fires `http_request`, then `extract_js_endpoints`, then begins probing
the API. Show the tools panel updating.

A `request_approval` for a recon-tier action fires. Operator hits Enter to
approve. Approval gate animation.

**VO:** "First the agent surveys the surface — discovers the endpoints,
notes the input shapes. The approval gate fires for anything intrusive; we
approve recon-tier actions on sight."

The agent's notes panel updates with a new task:

```
HYPOTHESIS: /api/products/preview accepts external URLs.
Test for SSRF.
```

**VO:** "And there's the hypothesis. The agent flagged the preview endpoint
on its own."

---

### Scene 5 — Educational interlude (1:50–2:20)

**Visual:** Cut away to the SSRF explainer card from the Findings deck.
Animated diagram:

```
[Attacker] → [App] → [Internal target attacker can't reach directly]
```

Caption: *Server-Side Request Forgery — when an app fetches a URL on behalf
of a user, an attacker can pivot through it to hit internal targets.*

**VO:** "Quick primer for anyone new to SSRF. Server-side request forgery
happens when an app fetches a URL on behalf of a user. If there's no
allowlist, the attacker's URL becomes the server's URL — and now the
attacker can reach things they shouldn't. Cloud metadata endpoints. Internal
admin panels. Things behind your firewall."

---

### Scene 6 — The exploit (2:20–3:30)

Cut back to terminal. Agent crafts a payload:

```json
{ "imageUrl": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
```

`request_approval` fires — this is intrusive, requires operator confirmation.
Pause for a beat. Operator approves.

**VO:** "Here's where the approval gate matters. The agent wants to send a
request that targets the cloud metadata IP. That's intrusive — the gate
holds it until we approve."

Approval granted. Request sent. Response comes back. Cut to the response in
the tools panel. Obfuscation overlay redacts the credential-shaped strings:

```
{
  "Code": "Success",
  "AccessKeyId": "▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓",
  "SecretAccessKey": "▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓",
  ...
}
```

**VO:** "And there it is. The server fetched the metadata endpoint on our
behalf. Those redactions are Apex's obfuscation mode — keeps your demos and
your screenshots clean."

---

### Scene 7 — The finding (3:30–4:20)

Agent calls `document_finding`. Finding pop-card:

> **🔴 Critical · Server-Side Request Forgery**
> `POST /api/products/preview · imageUrl parameter`
> Unauthenticated SSRF allows fetching internal endpoints, including AWS IMDS.
> **Suggested fix:** allowlist `imageUrl` host against a configured list, OR
> route preview fetches through a hardened egress proxy that blocks RFC1918,
> link-local, and metadata IPs.

**VO:** "Severity. Evidence. Suggested fix. That's an actionable bug report
your developer can fix tomorrow morning."

---

### Scene 8 — The report (4:20–4:45)

Open the report viewer. Show the SSRF finding rendered. Quick scroll past
the request/response evidence.

**VO:** "Same finding, in the report. This is what gets attached to the PR
or filed in your tracker."

---

### Scene 9 — Outro (4:45–5:00)

**Visual:** Find the Bug outro stinger.

```
EPISODE 2 NEXT WEEK
WHAT BUG WILL WE PLANT?
```

**VO:** "Find the Bug, episode one. Subscribe — episode two drops next
week, and we're not telling you the bug."

---

## Editing notes

- Approval gate is the dramatic beat. Linger on it. Let the viewer see the
  prompt for two full seconds before approving.
- The educational interlude (Scene 5) can be lifted whole into other SSRF
  content. Worth investing in good animation here — it'll be reused.
- Don't over-explain the obfuscation overlay; one line of VO is enough.
  We're spotlighting it more in Demo 12.
- This episode sets the **Find the Bug** template. Future episodes (`vuln-bank`
  race condition, `vuln-llm` prompt injection, etc.) should follow the same
  9-scene structure exactly.
