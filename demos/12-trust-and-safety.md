# Demo 12 — Approval tiers, scope guards, strict mode — staying in your lane

**Length:** 4 minutes
**Format:** Screencast w/ voiceover, calm pace
**Audience:** CISOs, sec-eng leads, anyone evaluating Apex for org-wide use
**Series:** Standalone (trust-and-safety positioning)

---

## Purpose

Tell the trust-and-safety story without it being defensive or dry. The
video should answer the unspoken procurement question: *"How does this
not blow up in our face?"*

Three pillars:
1. **Approval tiers** — graduated autonomy.
2. **Scope guards / strict mode** — it physically cannot leave the lane.
3. **Obfuscation mode** — share screenshots and demos without leaking.

---

## Requirements

- **Asset:** `vuln-admin` — internal admin panel with intentional bugs and
  multiple linked subdomains, including some that are intentionally
  *out-of-scope* (like a CDN or marketing site). The out-of-scope hosts
  are what we'll have the agent try, and watch get blocked.
- **Scope-guard log capture:** the toast / log line that fires when an
  out-of-scope request is blocked. Capture cleanly.
- **Before/after pair** showing `/obfuscate` redacting PII (hostnames,
  emails, tokens) in a live screenshot.
- **Tier reference card** graphic — describe what each of the 5 tiers
  auto-approves.
- **Theme:** `apex` dark.

---

## Things to check first

- [ ] Approval-tier behavior is documented somewhere canonical and the
      narration matches it. Tier 1 = read-only recon; tier 5 = autopilot.
      If the actual tiers don't match this script, update the script —
      not the product.
- [ ] Strict mode actually blocks out-of-scope requests at the tool layer
      (not just complains). Confirm with a recent run.
- [ ] `/obfuscate on` redacts the things we say it redacts: emails,
      hostnames, AWS-shaped keys, JWT shapes, auth headers.
- [ ] The "blocked redirect" scenario in Scene 3 fires reliably — pick a
      target where the agent will *naturally* try to follow a redirect
      out of scope.
- [ ] No real customer data, no real hostnames, anywhere in the recording.
      This episode is about safety; it should *be* safe.

---

## Full script

### Scene 1 — Cold open (0:00–0:20)

**Visual:** Dark frame. White text:

```
"How do we know it stays in its lane?"

— every CISO ever
```

Cut to terminal.

**VO:** "Every CISO asks this. So let's go through it: how Apex stays in
its lane. Three pieces — approval tiers, scope guards, and obfuscation."

---

### Scene 2 — Approval tiers (0:20–1:30)

**Visual:** Tier reference card overlay:

```
TIER 1 · Read-only recon                  (auto)
TIER 2 · + Endpoint enumeration           (auto)
TIER 3 · + Non-destructive probes         (auto)
TIER 4 · + Authenticated testing          (auto)
TIER 5 · + Intrusive payloads (autopilot) (auto)
```

Cut to terminal:

```
/operator --target https://vuln-admin.local --tier 2
```

The agent works. Recon-tier actions stream through without prompting. Then
an enumeration probe wants to fire — auto-approved at tier 2. Then a probe
that's *intrusive* — approval gate fires, modal appears.

**VO:** "Five tiers. You pick how much autonomy you want. Tier 1 is
read-only recon. Tier 5 is autopilot — only run that on a target you own
and a stack you've thrown away. Most teams sit at tier 2 or 3 in
production. Anything above the tier requires a human to approve."

Approval modal lingers for 2s. Operator presses Y.

---

### Scene 3 — Scope guards (1:30–2:30)

Cut to a fresh session:

```
/pentest --target https://vuln-admin.local \
         --strict \
         --hosts vuln-admin.local \
         --ports 443
```

Agent works. Encounters a redirect to `cdn.partner-marketing.com`. Scope
guard fires:

> ⚠ Out of scope: cdn.partner-marketing.com (blocked by --strict)

Highlight that line.

**VO:** "Strict mode means the agent literally cannot make a request that
falls outside your allowlist. That redirect — to a marketing CDN — got
blocked at the tool layer. The agent doesn't need to *try* to be good;
the tools won't fire. This is the difference between 'pentest' and
'unauthorized testing of a third party.'"

Show another out-of-scope attempt — a port outside `443` — also blocked.

**VO:** "Same thing for ports. We allowed 443, the agent tried 8080
because it found a hint of an admin panel there, the tool said no."

---

### Scene 4 — Obfuscation mode (2:30–3:20)

Cut to a finding view, no obfuscation. Visible: an internal hostname,
the test user's email, an auth token in the request header.

**VO:** "This is the screenshot you took to share in Slack. Internal
hostname, test user email, auth token. Maybe you trust your Slack —
fine. But what about a screenshot in a public talk? Or an issue tracker?"

Type:

```
/obfuscate on
```

Toast: *Obfuscation mode enabled — sensitive values will be redacted.*

Same finding view, now redacted:

- Hostname → `▓▓▓▓▓▓▓▓.local`
- Email → `▓▓▓▓▓▓@▓▓▓▓▓▓▓▓.local`
- Auth header → `Authorization: Bearer ▓▓▓▓▓▓▓▓▓▓▓▓...`

**VO:** "One slash command. Hostnames, emails, tokens, JWTs — all redacted
in real time, including in copy-paste. You can leave it on by default. We
do."

---

### Scene 5 — The whole picture (3:20–3:50)

**Visual:** Three icons fade in side-by-side:

```
[Tiers]      [Scope]      [Obfuscation]
 graduated    physically    safe to share
 autonomy     bounded       output
```

**VO:** "Three pieces. Approval tiers cap *how much* the agent can do
without you. Scope guards cap *where* it can act. Obfuscation caps what
leaves your screen. Together, that's the safety story. Nothing here gets
in the way of running Apex on a real surface — but everything's in place
for the day someone asks how you control it."

---

### Scene 6 — Outro (3:50–4:00)

```
APEX · STAY IN YOUR LANE
docs.pensar.dev/apex/safety
```

**VO:** "Docs link in the description. If you need this written up for
your security review — it's there."

---

## Editing notes

- Calm pace throughout. This is not a hype video — it's a "we thought
  about this" video.
- Lower-third should never get loud or flashy. Keep visual chrome subtle.
- The before/after on obfuscation is the strongest moment for sec-eng
  viewers — let it land.
- If a CISO can pull a quote out of this video for an internal review,
  the video did its job. Make Scene 5's narration quotable.
- Reuse this video's footage in the docs page for safety. Same clips,
  same examples — keeps everything aligned.
