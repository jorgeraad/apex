# Demo 13 — Pensar in the pipeline: gating production deploys

**Length:** 6 minutes
**Format:** Multi-surface screencast (TUI → Console → CI)
**Audience:** Platform engineers, sec-eng, dev-leads
**Series:** Standalone

---

## Purpose

Show the end-to-end story for an org adopting Apex: local TUI for
exploration, Pensar Console for scheduling and history, headless CLI in
CI. The viewer should walk away seeing how Apex fits into the SDLC, not
just sitting beside it.

This is the natural follow-up to Demo 06 (PR-time pentest), with more
emphasis on Console and on long-running surface coverage.

---

## Requirements

- **Asset: CI repo** (reuse from Demo 06).
- **Pensar Console access** — real instance preferred. Mocked screens are
  acceptable but only as a fallback; flag clearly in the description if so.
- **Slack workspace** with a `#sec-pensar-alerts` channel for the webhook
  drop.
- **Sample finding history** in Console — at least 4 weeks of data so the
  trend chart looks real.
- **Login flow recording:** capture the device-flow login once, reuse.
- **Theme:** `apex` dark.

---

## Things to check first

- [ ] Console instance is reachable and seeded with realistic-looking demo
      data. Names, hostnames are all `*.demo.local`.
- [ ] Slack webhook fires on the real Slack channel. Test before recording.
- [ ] CI repo's preview deploy URL still works (pulled from Demo 06's
      asset).
- [ ] Login → workspace selection flow is smooth on a clean machine. If
      it stalls or shows an error toast, fix it before recording.
- [ ] The "schedule a nightly run" action in Console is working — that's
      the hero feature for this video.
- [ ] Production-deploy gating example uses a clearly fake "production"
      environment. Don't risk implying any real production system was
      gated by the demo.

---

## Full script

### Scene 1 — Cold open (0:00–0:20)

**Visual:** Three windows tile across the screen:

```
[TUI]    [Console]    [CI]
```

**VO:** "Three surfaces. One pentest. Today we're walking through how a
team uses Apex across the whole development cycle — local exploration,
scheduled scans, and pipeline gating."

---

### Scene 2 — Local exploration finds something (0:20–1:30)

Cut to TUI. Operator session against a staging URL.

```
/operator --target https://staging.demo.local
```

Quick run, agent finds an IDOR. Finding card visible.

**VO:** "Start in the TUI. A security engineer is exploring a new feature
in staging, finds an IDOR. So far, this is the workflow we've shown in
other videos. The new part is what happens next."

---

### Scene 3 — Login + handoff (1:30–2:30)

Type:

```
/login
```

Device-flow login dialog appears. Code, browser, workspace pick. Capture
authentic but quick.

**VO:** "`/login` connects this session to Pensar Console — our
cloud-hosted edition. Once connected, today's session lives in Console
as well as locally. That means findings, history, and the agent's full
trace are visible to the rest of the team."

Cut to Console. Today's session is at the top of the list, IDOR finding
visible.

**VO:** "Same finding. Now in Console."

---

### Scene 4 — Schedule it (2:30–3:30)

In Console, navigate to the project. Click "Add scheduled run." A panel
opens — target URL, frequency, threat model selector.

Set:

```
Target: https://staging.demo.local
Frequency: Nightly @ 02:00 UTC
Threat model: ./threat-model.md (latest version)
Auth: stored test-user credentials
Notify on: any high or critical
```

Save. The schedule appears in a list.

**VO:** "Schedule it. Nightly run against staging, with the same threat
model the team committed in the repo. Notifications go out on anything
high-or-critical. This is how you go from 'we ran a pentest once' to
'we run a pentest every night.'"

---

### Scene 5 — The trend (3:30–4:15)

Click into the project's history view. A trend chart shows findings over
the last four weeks: 9 open, 12 fixed, 3 net-new last week.

**VO:** "Trends. This is what the security review meeting actually wants
— net-new, time-to-fix, and what's still open. Each one of these is a
finding the agent surfaced; each fixed one is a regression Apex confirmed
went away."

---

### Scene 6 — Pipeline gate (4:15–5:15)

Cut to a CI repo PR. The PR is in `prod-deploy` flow — the workflow
includes a Pensar gate before deployment.

```yaml
- name: Production gate (Apex)
  run: |
    pensar pentest \
      --target $PROD_PREVIEW_URL \
      --threat-model @./security/threat-model.md \
      --severity-threshold high \
      --max-duration 600
```

Run it. A new high-severity finding appears mid-run. The CI step exits
non-zero. Deploy is blocked.

**VO:** "Pipeline gate. Same agent, headless, in CI. We added a severity
threshold — anything 'high' or above blocks the deploy. The team gets a
PR comment with the finding. The deploy waits."

---

### Scene 7 — Slack alert (5:15–5:40)

Cut to Slack. A message arrives in `#sec-pensar-alerts`:

> 🚨 *Production deploy blocked — high-severity finding*
> Project: demo-prod · Finding: SQL Injection in `/api/orders`
> [Open in Console ↗]  [PR #248 ↗]

**VO:** "And the team knows. Slack drop, link to Console, link to the
PR. Same loop you'd already run for any other CI failure — just with a
real pentest gating it."

---

### Scene 8 — Outro (5:40–6:00)

**Visual:** Three windows return, now with status badges:

```
[TUI ✓]    [Console ✓]    [CI ✓]
```

**VO:** "Local. Cloud. Pipeline. Same agent everywhere. That's how teams
run Apex at org scale. Console is at pensar.dev/console — link in the
description."

---

## Editing notes

- This video is the closest the series gets to a "product walkthrough."
  Don't apologize for it — but keep cuts tight. No screen lingers more
  than 6 seconds without something happening.
- Console screens should match your current production styling. If the
  product changes, re-record this episode rather than letting it drift.
- The Slack drop is satisfying — give it a beat to land before cutting.
- If you can't show real Console, replace Scenes 3–5 with Console
  mockups *and clearly say so on screen*: "Console screens are mockups
  in this version of the video — Console is in beta." Trust matters more
  than polish.
