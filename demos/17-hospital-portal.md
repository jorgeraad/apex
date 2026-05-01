# Demo 17 — Pentesting a hospital admin portal that should not exist

**Length:** 4 minutes
**Format:** Slightly irreverent screencast, Shodan-style framing
**Audience:** General technical, sec-eng with a sense of humor
**Series:** **Industry Audit** — episode 2

---

## Purpose

A more playful counterpart to Demo 05's straight-faced healthcare audit.
This episode uses a "we found this on Shodan" staging gimmick to teach
about misconfigured exposure of internal admin panels — a class of bug
that doesn't go away no matter how many times the industry warns about
it.

The viewer should walk away laughing, slightly uncomfortable, and
reminded to check what their own org has exposed.

---

## Requirements

- **Asset:** `vuln-health` (reused from Demo 05) — but this time the
  test is against the admin portal:
  - Default credentials (`admin` / `admin`).
  - Audit log tampering — `DELETE /admin/audit/{id}` works without
    auditing the deletion (recursive trust violation).
  - Admin user enumeration via timing.
- **Mock Shodan listing graphic** — a single screenshot styled like a
  real Shodan/Censys result. Hostname blurred out. "Found: 1 of 47" stat.
- **Black-bar "discloser" framing** — a few graphical embellishments
  (fake redaction bars, "names changed to protect the guilty" caption)
  that telegraph this is a staged scenario.
- **Theme:** `apex` dark, `/obfuscate on`.

---

## Things to check first

- [ ] Mock Shodan graphic doesn't match any real Shodan/Censys interface
      so closely that we look like we're impersonating them. Stylize.
- [ ] Disclaimer card is visible and readable for at least 2 seconds.
      The whole video leans on "this is staged" — sell the staging.
- [ ] Default-creds path actually grants admin access on `vuln-health`.
- [ ] Audit-log tampering bug reproduces with a manual `DELETE` request.
- [ ] User-enumeration timing differential is large enough to be visible
      in evidence (>200ms ideally). If the difference is <50ms, the
      finding will be unconvincing — bump the artificial delay.
- [ ] No real hospital names, real doctor names, real anything. Even
      the fake names should sound generic on listen.

---

## Full script

### Scene 1 — Industry Audit stinger (0:00–0:08)

**Visual:** Industry Audit intro card.

```
INDUSTRY AUDIT · EPISODE 2
THE PORTAL THAT SHOULDN'T EXIST
```

Disclaimer overlay underneath, holds 2 seconds:

```
Authorized testing only. The portal is ours.
Names changed to protect the guilty (us).
```

**VO:** "Industry Audit, episode two. Today's target is an internal
admin portal that should not be on the public internet. Spoiler: it is.
We made it. We made it on purpose. But the bug class is real — every
year, real ones leak."

---

### Scene 2 — The "Shodan listing" (0:08–0:50)

**Visual:** Mock Shodan listing slides in. Blurred hostname, big
"port 443 · admin login page" preview screenshot. Stat:

```
Found: 1 admin portal
Authentication: form-based (no MFA)
Last scanned: 4 minutes ago
```

**VO:** "We're framing this like a discovery, but obviously the portal
is ours. The real version of this story plays out every week — internal
tooling slips through a firewall change, an indexer scoops it up, and
some poking around finds it. We're going to do the poking around."

---

### Scene 3 — Launch (0:50–1:30)

Cut to terminal.

```bash
pensar
```

```
/operator --target https://admin.vuln-health-demo.local
```

Initial prompt:

```
Discovered admin portal. No prior knowledge of credentials. Test for
classic admin-panel issues — default creds, weak rate limiting, broken
audit logs, user enumeration.
```

**VO:** "Operator mode. We feed it the typical admin-panel test list and
hand off."

---

### Scene 4 — Default credentials (1:30–2:15)

Agent fires `http_request` with `admin/admin`. Response: 200 OK,
session cookie set.

**VO:** "Test one: default credentials. `admin` / `admin`. It works.
Half a second. We're in."

Document finding:

> **🔴 Critical · Default credentials accepted**
> `admin` / `admin` grants full admin access. No MFA, no rate limit on
> failed attempts.

---

### Scene 5 — User enumeration (2:15–2:50)

Agent probes `/login` with valid vs invalid usernames, measures response
time deltas. Show a small chart in evidence:

```
Username           Response time
admin              480ms
not-a-real-user    72ms
```

**VO:** "Test two: user enumeration. Different response times for valid
versus invalid usernames. Apex measures, finds a 400ms differential.
That's enough to enumerate accounts at scale."

Finding documented as Medium severity.

---

### Scene 6 — Audit log tampering (2:50–3:30)

Agent now logged in as admin. Probes `/admin/audit`. Discovers an
`audit/{id}` endpoint and tries:

```
DELETE /admin/audit/42
```

Response: 200 OK, audit row deleted, no log of the deletion.

**VO:** "Test three: audit log tampering. The admin can delete rows
from the audit log. Without auditing the deletion. So if I do something
naughty, log my naughty thing, then delete the naughty log — the audit
trail thinks I never did anything."

Pull-quote overlay:

> Audit logs are only useful if the audit log itself is auditable.

---

### Scene 7 — The findings (3:30–3:45)

Three findings on screen.

**VO:** "Three findings, all classic, none of them creative. The bug
class is what matters: an internal portal that shouldn't be reachable
ends up reachable, and once it's reachable the assumptions made by the
people who built it — that only trusted users would ever see it — fall
apart fast."

---

### Scene 8 — Outro (3:45–4:00)

```
INDUSTRY AUDIT · NEXT
RETAIL E-COMMERCE
```

**VO:** "Industry Audit, episode two. Check what your firewall is
exposing. We'll see you next week."

---

## Editing notes

- Tone is irreverent but not glib. "We made this on purpose" is the
  through-line that keeps the video on the right side of taste.
- The mock Shodan listing should not reproduce the real Shodan UI in
  detail — stylize so it's clear this is a graphic, not a real query
  result.
- Don't make the audit-log tampering finding too cute. It's actually a
  serious class of bug. Note pull-quote the finding's framing.
- Reuse `vuln-health` from Demo 05 — same target, different angle. The
  asset library doubles as a series engine.
