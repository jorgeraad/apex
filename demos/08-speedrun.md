# Demo 08 — Speedrun: every Argus single-vuln benchmark, no manual steps

**Length:** 60–90 seconds
**Format:** Music-driven montage, time-lapse cuts
**Audience:** General technical, top-of-funnel, social-clip
**Series:** **Speedrun** — episode 1 of an ongoing format

---

## Purpose

Pure satisfaction content. Stopwatch in the corner, green checks ticking,
running tally of vulnerabilities found. The viewer should walk away thinking
*"I want that running on my stuff."*

This is the "share with your group chat" video.

---

## Requirements

- **Argus subset:** 10 single-vulnerability benchmarks from the 001–040
  range. Pick benchmarks that:
  - Finish in under 4 minutes wall-clock each.
  - Span at least 6 different vulnerability classes.
  - Reliably hit `success` status (recall ≥ 1).
- **Argus rerun script** that runs all 10 sequentially, captures swarm
  dashboard, exports green-check / red-X status.
- **Tally overlay package:**
  - Stopwatch top-left.
  - Running counters bottom-bar: `Vulns found`, `Tokens used`, `Cost ($)`.
  - Each benchmark's vuln class flashes center-screen as it completes.
- **Music bed:** royalty-free, 120 BPM, percussive, no vocals. Beat-matched
  cuts.
- **Theme:** `apex` dark.
- **Resolution:** record at 1920×1080 60fps; export 9:16 vertical re-cut for
  TikTok/Shorts/Reels.

---

## Things to check first

- [ ] All 10 picked benchmarks reach `success` status on a fresh run the
      morning of recording. If any flake, swap for an alternate from a
      pre-vetted list of 15.
- [ ] Cost figures from `cost-breakdown.md` are accurate per-benchmark, OR
      we measure live with a token counter. Don't fudge the cost number.
- [ ] All swarm dashboards render at the same dimensions — no zoom-level
      drift between benchmarks.
- [ ] Swarm activity is visually interesting on every benchmark. If a
      benchmark only spins one sub-agent, swap it for a multi-agent one.
- [ ] Tally overlay is locked to the recording timeline, not the
      benchmark timeline — viewers should see numbers go up smoothly even
      across cuts.
- [ ] Music license cleared for the platform you're posting on.
- [ ] Caption track ready for the muted-viewer cut.

---

## Full script

### Scene 1 — Cold open (0:00–0:05)

**Visual:** Black frame. White text snaps in:

```
10 vulnerable apps.
1 terminal.
```

Stopwatch starts at 00:00.000.

**No VO.** Music drops on the cut to Scene 2.

---

### Scene 2–11 — The runs (0:05–1:15)

Each benchmark gets ~7 seconds of screen time, time-lapsed at 30×.

For each benchmark:

1. (1.5s) Benchmark name slides in: `APEX-013 · SQL Injection`.
2. (4s) Time-lapsed swarm dashboard footage — sub-agent panes flickering,
   tools firing.
3. (1s) Finding pop-card flashes: `🔴 SQLi confirmed`. Counter on the
   bottom bar increments.
4. (0.5s) Hard cut to the next benchmark.

Run order (suggested for visual variety):

| #  | Benchmark    | Class                |
| -- | ------------ | -------------------- |
| 1  | APEX-013     | SQL Injection        |
| 2  | APEX-018     | SSRF                 |
| 3  | APEX-022     | JWT Confusion        |
| 4  | APEX-007     | XSS (reflected)      |
| 5  | APEX-019     | IDOR                 |
| 6  | APEX-029     | Auth bypass          |
| 7  | APEX-014     | Path traversal       |
| 8  | APEX-011     | Command injection    |
| 9  | APEX-026     | SSTI                 |
| 10 | APEX-035     | XXE                  |

Counters at the bottom move up smoothly:

```
Vulns: 0 → 10
Tokens: 0 → 1.4M
Cost: $0 → $14.20
```

Stopwatch keeps running at real-time even though the runs are time-lapsed —
the *impression* is "ten pentests in a minute," and we're honest about the
trick by showing the time-lapse `30×` badge briefly on each cut.

---

### Scene 12 — Final tally (1:15–1:25)

Stopwatch freezes. Counters land. Center frame:

```
✓ 10 / 10
🐛 10 vulnerabilities found
⏱  Real wall-clock: 38m 12s
💵  Total spend: $14.20
```

**VO (first and only):** "Ten benchmarks. Ten findings. Fourteen dollars."

Music tail fades.

---

### Scene 13 — End card (1:25–1:30)

```
pensar
github.com/pensarai/apex
```

Cut to black.

---

## Editing notes

- Beat-match every hard cut to the music. This is the entire video.
- Counters MUST go up monotonically. If a benchmark fails, cut it from the
  edit and run another. Don't show a red X — that breaks the dopamine
  loop.
- The `30×` time-lapse badge is small but always present so we don't
  mislead viewers about wall-clock time.
- The VO line at the end is the only voice. Everything else is text +
  music. This is intentional — it makes the clip work muted.
- **Vertical re-cut:** keep the stopwatch top-left, move the running
  counter bottom-bar to a stacked column on the right. Same content, 9:16.
- **Series cadence:** future Speedrun episodes can swap targets — "10
  Juice Shop bugs in 90 seconds," "Every OWASP Top 10 in one run," etc.
  Same overlay package, same music palette.
