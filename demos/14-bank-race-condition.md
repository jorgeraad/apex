# Demo 14 — Banking app, race conditions, $0 to $10M in five minutes

**Length:** 5 minutes
**Format:** Suspenseful screencast w/ animated balance overlay
**Audience:** General technical, pentesters, anyone who likes a good story
**Series:** **Find the Bug** — episode 3

---

## Purpose

Tell a story. Race conditions are a class of bug that's hard to demo in
words and easy to demo in numbers. Show a balance going from $0 to
$10,000,000 in 30 seconds and the point lands.

This episode is the most "fun" of the Find the Bug series — lean into
the heist framing.

---

## Requirements

- **Asset:** `vuln-bank` — Flask banking app with:
  - Two test accounts: `alice` (starting balance $0) and `bob` (starting
    balance $10,000,000).
  - A `/transfer` endpoint with non-atomic balance check + decrement, so
    parallel requests can each pass the balance check before any one of
    them decrements.
  - A simple web UI at `/dashboard` showing live balances. The UI auto-
    refreshes every second so we can watch the number climb on camera.
- **Animated balance overlay** — a giant ticker showing alice's balance,
  matched to the dashboard but at full screen-resolution.
- **Pentester narration tone** — calm, slightly amused. This is a heist
  scene; play it cool.
- **Theme:** `apex` dark, `/obfuscate` off (the values are the point).

---

## Things to check first

- [ ] Race condition is reproducible. Confirm with a manual `xargs -P 50`
      attack before recording. Some hosts will randomize timing enough that
      the race rarely lands — bench the demo machine.
- [ ] Dashboard balance refreshes smoothly. Stale-balance UI is the most
      common visual fail.
- [ ] Reset script returns alice to $0 and bob to $10,000,000 cleanly.
- [ ] Apex actually drives this exploit on a dry run. If the agent doesn't
      get the race-condition idea, prepend a hint to the prompt — but try
      without first.
- [ ] All hostnames and routing IDs in the recording are obviously fake
      (e.g., `bob@vuln-bank-demo.local`). This is the kind of demo that
      gets clipped and shared — make sure any clip is unmistakably a demo.
- [ ] Disclaimer card at the start is present. "Authorized testing only.
      This bank does not exist."

---

## Full script

### Scene 1 — Find the Bug stinger + disclaimer (0:00–0:10)

```
FIND THE BUG · EPISODE 3
RACE CONDITION

Authorized testing only. This bank does not exist.
```

**VO:** "Find the Bug, episode three. Today, a bank — that doesn't
exist."

---

### Scene 2 — Meet alice (0:10–0:50)

Browser: dashboard view. Alice is logged in. Big balance: $0.

**VO:** "Alice has zero dollars. Bob has ten million. They both bank at
Vuln-Bank, a service we made up for this video. The transfer endpoint
between accounts has a bug. We didn't tell Apex what the bug is."

Show the transfer UI in action — alice moves $5 from herself (which
she doesn't have) to bob. Server returns "Insufficient funds." Normal.

**VO:** "Normal transfers fail. So far so good."

---

### Scene 3 — Launch operator (0:50–1:30)

```bash
pensar
```

```
/operator --target https://vuln-bank.demo.local
```

Initial prompt:

```
We have an authenticated session as 'alice' (balance: $0). The /transfer
endpoint is the most interesting target. Look for any way to move funds
that shouldn't be possible. Race conditions are in scope.
```

**VO:** "We give the agent a hint that race conditions are in scope —
they're notoriously hard to discover from a clean cold start. But we
don't tell it where to look."

---

### Scene 4 — Recon + hypothesis (1:30–2:20)

Agent tests `/transfer` with various values. Behavior is correct under
serial requests.

Reasoning callout:

> "Sequential requests behave correctly. The check-then-decrement pattern
> is suspicious. If the check is non-atomic, parallel requests might each
> see the same starting balance."

Approval gate fires for a parallel-request probe.

**VO:** "There's the hypothesis. The agent figured out the shape of the
bug from observing serial behavior — a non-atomic check-then-decrement.
Now it wants to fire parallel requests."

---

### Scene 5 — The exploit (2:20–3:30)

Operator approves. Agent fires:

```
50× POST /transfer
{ "from": "bob", "to": "alice", "amount": 200000 }
```

(Note: this assumes bob's session was acquired, OR a second-account TOCTOU
where alice can claim funds via a refund flow. Adjust to whatever bug
`vuln-bank` actually plants.)

Cut to dashboard. Balance overlay zooms in. Numbers tick:

```
$0 → $200,000 → $400,000 → $1,200,000 → $4,000,000 → $9,800,000 → $10,000,000
```

The balance climbs visibly. Sub-second pacing. Drop in tense beat-music
overlay (royalty-free).

**VO (during the climb):** "Fifty parallel requests. Each one sees a
starting balance high enough to allow the transfer. None of them have
seen the others yet. By the time the dust settles…"

Alice's balance: **$10,000,000**. Bob's balance: **$0**.

Music drops on the freeze frame.

**VO:** "Ten million."

---

### Scene 6 — The finding (3:30–4:15)

Agent calls `document_finding`:

> **🔴 Critical · Race condition in `/transfer`**
>
> Non-atomic balance check allows TOCTOU exploitation via parallel
> requests. 50 concurrent requests successfully transferred more value
> than the source account held.
>
> **Suggested fix:** wrap balance check + decrement in a single database
> transaction with appropriate isolation level (`SERIALIZABLE` or
> `SELECT … FOR UPDATE`).

**VO:** "Severity: critical. Evidence: a balance that shouldn't exist.
Suggested fix: actually use a transaction. Half of all race-condition
bugs would not exist if developers learned `SELECT FOR UPDATE` on day
one."

---

### Scene 7 — Reset (4:15–4:40)

Cut to a quick `make reset` in the terminal. Dashboard refreshes — alice
back to $0, bob back to $10,000,000.

**VO:** "Reset. The bank is fine. The bug is documented. Fix shipped
before the next episode airs."

---

### Scene 8 — Outro (4:40–5:00)

Find the Bug outro card.

```
EPISODE 4
WE'RE NOT TELLING YOU THE BUG
```

**VO:** "Find the Bug, episode three. Subscribe."

---

## Editing notes

- The balance climb in Scene 5 is the entire video. Spend the post-
  production time there. Big readable digits, smooth tick animation, beat
  match to a music drop.
- Tone: calm pentester. Not gleeful. The exploit is interesting *because*
  it's mundane in retrospect — that's the point.
- Disclaimer in Scene 1 must be visible long enough to read. Two full
  seconds. Preferably also in the description.
- This is a high-share video. Plan a vertical re-cut: just Scenes 2, 5,
  6, 8. About 90 seconds.
- If the live race condition refuses to fire on recording day, do not
  fake it. Reschedule.
