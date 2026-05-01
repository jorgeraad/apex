# Demo 01 — Cold start to first finding in 90 seconds

**Length:** 90 seconds
**Format:** Screencast w/ voiceover, brisk MIDI music
**Audience:** Developers
**Series:** Standalone (works as the canonical "first watch")

---

## Purpose

Prove that a developer with zero prior context can install Apex, run a real
pentest, and see a real finding inside 90 seconds. This is the canonical
top-of-funnel video — the one we link from the README, the homepage, and the
"what is this" tweet.

The viewer should walk away thinking: *"Wait, that's it?"*

---

## Requirements

- **Recording surface:** clean macOS terminal (or Linux/Kali for variety later),
  font: a recognizable mono (JetBrains Mono / Berkeley Mono), 14–16pt.
- **Asset:** `vuln-shop` — Node/Express + Postgres demo e-com app, planted SQLi
  on `/search?q=`. Reachable at `https://vuln-shop.local` via local DNS or
  Tailscale.
- **Anthropic API key** (or pre-configured Pensar Console token) — keep the
  paste off-frame and use a redacted overlay.
- **Demo overlay package:** install lower-third, command ribbon, finding pop-card.
- **Music bed:** royalty-free, 90 BPM, no vocals.
- **Theme:** `apex` dark.
- **Resolution:** 1920×1080, 60fps, terminal occupies center 70% of frame.

---

## Things to check first

- [ ] `vuln-shop` is up and reachable. SQLi payload (`' OR 1=1 --`) works in a
      quick manual smoke test.
- [ ] `vuln-shop` resets to a known seed state between takes (script: `make reset`).
- [ ] No real API keys, hostnames, or PII visible. Run `/obfuscate on` first
      take just to confirm the overlay works, then turn off for cleaner footage
      since `vuln-shop.local` is already fake.
- [ ] Findings registry is reachable in the TUI — tap into the report viewer
      after the run to confirm the SQLi finding renders with title, severity,
      evidence (request + response).
- [ ] Recording machine has the swarm dashboard responding crisply — sub-agent
      tiles should populate within 5 seconds of `/pentest` start.
- [ ] Total runtime against `vuln-shop` is under 3 minutes (we'll time-lapse
      the middle, but the agent must actually finish in one go for a good cut).
- [ ] Overlay package version-matches the rest of the series (no font drift).

---

## Full script

### Scene 1 — Cold open (0:00–0:08)

**Visual:** Black frame. Single line types in monospace, center:

```
how long does a real pentest take?
```

**VO (calm, conversational):** "How long does a real pentest take?"

**Cut to:** clean Terminal.app window, empty prompt. Lower-third: *Pensar Apex.
Cold install. No prep.*

---

### Scene 2 — Install (0:08–0:20)

**On screen, typed at human speed:**

```bash
curl -fsSL https://pensarai.com/install.sh | bash
```

Install completes — show the last 3 lines of output, including
`✓ Installed pensar to /usr/local/bin/pensar`.

**VO:** "One install line. macOS, Linux, Windows — pick your shell."

---

### Scene 3 — First launch (0:20–0:35)

**Type:**

```bash
pensar
```

TUI boots. Responsible Use disclosure appears. Enter is pressed.

**VO:** "First launch asks you to agree to responsible use. Authorized testing
only — that's the deal."

Provider Manager appears. Paste API key (paste happens off-frame, key blurred
on screen by overlay). The key field briefly shows `sk-ant-•••••••••` then
collapses.

**VO:** "Paste an API key from any provider — Anthropic, OpenAI, Bedrock, or a
self-hosted vLLM model. We're using Claude here."

---

### Scene 4 — Run the pentest (0:35–0:55)

**Type:**

```
/pentest --target https://vuln-shop.local
```

The wizard auto-skips because `--target` is set. Cut to operator view spinning
up the swarm.

**VO:** "One slash command. One target. Apex handles the rest."

Quick zoom-in on the swarm dashboard tile grid as 4 sub-agents come online.

---

### Scene 5 — Time-lapse the work (0:55–1:15)

**Visual:** 4× speed. Sub-agent panes scroll. Tools fire — `http_request`,
`extract_js_endpoints`, `test_endpoint_variations`. Tasks tick from PLAN
through TEST.

**VO (over the time-lapse):** "Underneath, a swarm of specialized agents is
mapping the attack surface, splitting up targets, and probing each one through
a strict seven-step methodology. You'll see how it works in another video."

A finding pop-card slides in from the right edge:

> **🔴 Critical · SQL Injection**
> `GET /search?q=' OR 1=1 --`
> Database error leaks `users.password_hash`

**VO:** "There's the punchline."

---

### Scene 6 — The report (1:15–1:25)

Cut from time-lapse back to real-time. Agent posts the summary message:

```
Pentest complete. 1 critical, 2 high, 0 medium, 1 low.
Report saved to ./pentest-report.md
```

Open the report viewer dialog. Scroll once to show: finding title, severity,
evidence (request + response), suggested fix.

**VO:** "Severity, evidence, and a suggested fix. This is the actionable part —
you can hand it straight to the developer who shipped the bug."

---

### Scene 7 — Outro (1:25–1:30)

**Visual:** Terminal fades. Card:

```
pensar
github.com/pensarai/apex
```

**VO:** "One install. One command. Real findings. Apex is open source — link
in the description."

End card holds 2 seconds. Cut to black.

---

## Editing notes

- Cut hard between scenes — no fades except the very last.
- Music drops on the finding pop-card reveal (Scene 5). Beat-match the cut.
- Caption every VO line — many viewers watch muted.
- Lower-third name plate stays on for the first 5 seconds only.
- Export a 60s vertical re-cut for social: drop Scenes 1 and 7, tighten install.
