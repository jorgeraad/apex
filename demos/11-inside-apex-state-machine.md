# Demo 11 — Inside Apex #2: the 7-step pentest state machine

**Length:** 5 minutes
**Format:** Animated explainer w/ live cutaways and a counter-example
**Audience:** Sec-eng, pentesters, agent-architecture viewers
**Series:** **Inside Apex** — episode 2

---

## Purpose

Make the case for the strict
PLAN → VERIFY → PREPARE → TEST → EXPLOIT → DOCUMENT → FINISH
loop in `TargetedPentestAgent`. Rooted in
[`decisions/PDR-004-pentest-methodology.md`](../decisions/PDR-004-pentest-methodology.md).

The viewer should walk away with: *"Without this, agents drift. With this,
output is consistent."*

The differentiator in this episode is the **live counter-example** — we run
the same pentest twice, once with the loop enforced and once without, and
show the difference.

---

## Requirements

- **Asset:** `vuln-shop` with at least 3 plantable findings (SQLi, IDOR,
  weak auth) so we can see whether each variant documents them.
- **Forked Apex build** with the strict state machine disabled (e.g., via a
  build flag `APEX_DISABLE_STATE_MACHINE=1`). This is a temporary fork for
  the demo only — do not ship it.
- **State machine animation:** the seven phases as a closed-loop diagram,
  with arrows showing forward-only progression and an indicator that
  pulses when the agent is in each phase.
- **Comparison overlay:** side-by-side findings tally that builds over the
  course of each run.
- **Theme:** `apex` dark.

---

## Things to check first

- [ ] Forked build is signed off as "demo-only" — README on the branch
      makes clear it must never ship. We don't want this to leak.
- [ ] Both runs (enforced vs disabled) reliably produce different outcomes
      — at least one finding missed or undocumented in the disabled run on
      a dry pass. Otherwise the counter-example flops.
- [ ] State-machine animation phases match the actual prompt-encoded loop
      exactly. Read the source of truth in `src/core/agents/specialized/pentest/`
      and confirm before recording.
- [ ] Both runs use the same model, same temperature, same target, same
      seed where possible. The only variable should be the loop.
- [ ] Findings registry is reset between the two runs.
- [ ] Pre-record the comparison overlay so we can drop it on top of the
      footage in post — capturing it live across two runs is fragile.
- [ ] Don't ship the forked build to npm or any registry, even by accident.

---

## Full script

### Scene 1 — Cold open (0:00–0:20)

**Visual:** State machine diagram appears: a 7-node loop.

```
PLAN → VERIFY → PREPARE → TEST → EXPLOIT → DOCUMENT → FINISH
```

**VO:** "Inside Apex, episode two. Today: the seven-step state machine
inside `TargetedPentestAgent`. Why we make the agent march through it in
order. And what happens when we don't."

---

### Scene 2 — The drift problem (0:20–1:15)

**Visual:** Animated agent stream of consciousness — text streaming as if
from an LLM:

```
"I've found a few interesting endpoints. The auth one
looked solid. I'm going to declare this complete."
```

A red ✗ slams onto the screen.

**VO:** "Here's the failure mode we kept hitting in early versions.
Free-form pentest agents drift. They stop early. They skip the hard steps
— specifically exploitation and documentation, which are exactly the
steps that make a finding actionable. They hallucinate completion. 'I have
finished testing this endpoint.' And then they walk away."

Pull-quote:

> "Soft guidance is routinely ignored under the pressure of a long context
> window." — PDR-004

---

### Scene 3 — The seven steps (1:15–2:30)

**Visual:** The state machine animates one phase at a time. For each
phase, a one-line description and a snippet of what the agent does:

| Phase    | What the agent does                                              |
| -------- | ---------------------------------------------------------------- |
| PLAN     | Lists candidate test cases and orders them                       |
| VERIFY   | Confirms scope and target reachability                           |
| PREPARE  | Stages the environment, opens browser, sets headers              |
| TEST     | Issues the actual probe                                          |
| EXPLOIT  | If a probe lands, follows it through to demonstrable impact      |
| DOCUMENT | Writes the finding to the registry with evidence                 |
| FINISH   | Declares this objective complete and returns                     |

**VO:** "Plan, then verify. Set up. Test. If something hits, exploit it
through to impact. Document with evidence. Then — and only then — finish.
The agent literally cannot declare completion without reaching FINISH.
That's a structural constraint, not a polite request."

---

### Scene 4 — Live: enforced run (2:30–3:30)

Cut to terminal.

```bash
pensar pentest --target https://vuln-shop.local
```

Time-lapse the run. Phase indicator overlay (top-right) ticks through the
loop for each sub-agent. Findings populate.

End-of-run summary:

```
✓ 3 findings documented
  - 🔴 Critical SQLi
  - 🟠 High IDOR
  - 🟡 Medium weak auth policy
```

**VO:** "Enforced loop. Same target we'll use in a moment. Three findings,
all with evidence and suggested fixes. Documentation is mandatory, so
nothing leaves the agent without it."

---

### Scene 5 — Live: counter-example (3:30–4:15)

Cut to terminal in the forked build.

```bash
APEX_DISABLE_STATE_MACHINE=1 pensar pentest --target https://vuln-shop.local
```

Run it. The agent moves faster but more chaotically. End-of-run summary:

```
✓ 2 findings documented
  - 🔴 Critical SQLi (but evidence: empty)
  - 🟠 High IDOR (but no exploit path shown)

✗ Weak auth policy: detected in chat but not documented
```

Comparison overlay slides in:

```
        Loop ON   Loop OFF
Findings    3        2
Evidence  full     thin
Skipped   none    weak-auth
```

**VO:** "Same target, same model, loop disabled. Two findings filed
instead of three — the agent noticed weak auth and never wrote it up.
And the two it did file have thin evidence — no demonstrated exploit. This
is what 'agentic drift' looks like in practice."

---

### Scene 6 — Why this matters (4:15–4:45)

**VO:** "If your tool is going to claim coverage, that claim has to be
auditable. You can't audit something the agent merely *intended* to
document. The state machine is the difference between 'we tested this'
and 'we tested this *and* we have the receipts.'"

---

### Scene 7 — Outro (4:45–5:00)

```
INSIDE APEX · EPISODE 2
NEXT: COST OF A PENTEST
```

```
decisions/PDR-004-pentest-methodology.md
```

**VO:** "Inside Apex, episode two. Episode three is about money: what
does a real pentest actually cost?"

---

## Editing notes

- The counter-example is the value of this video. Don't shortchange it.
  If it's flaky, re-shoot until it lands cleanly. It's the most
  persuasive moment in the entire Inside Apex series.
- Be honest in the comparison overlay. If the loop-off run sometimes
  matches the loop-on run (LLMs are stochastic), say so explicitly:
  "On a good day the free-form run gets all three. On a bad day, it
  gets one. Consistency is the point."
- The "demo-only fork" framing should be visible — we do not want
  someone screenshotting this and claiming Apex secretly has a flag to
  disable safety. Add an on-screen disclaimer in Scene 5.
- Phase indicator overlay is reused from Demo 03 (`/threat-model`). Same
  package, same colors.
