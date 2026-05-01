# Demo 02 — Apex vs Argus: scoring 34 of 60 benchmarks live

**Length:** 4–5 minutes
**Format:** Narrated screencast w/ data-vis interludes
**Audience:** Security engineers, CISOs, technically-minded buyers
**Series:** Standalone (credibility piece)

---

## Purpose

Establish credibility with hard, public numbers. Apex was scored against the
60-benchmark Argus suite and got 100% recall on 34 of them — that's a fact we
can prove with the public CSV in `benchmarks/argus/`.

This is the "is this for real?" video. The viewer should walk away believing
Apex is a serious tool with serious evaluation behind it.

---

## Requirements

- **Public data:** `benchmarks/argus/consolidated-results.csv`,
  `consolidated-report.md`, `cost-breakdown.md` — already in the repo.
- **Argus subset:** 3 hand-picked benchmarks that finish in under 4 minutes
  each, span at least two vulnerability classes (e.g., SQLi, SSRF, JWT
  confusion), and produce visually interesting swarm activity.
- **Argus rerun script:** wraps `pensar pentest` against each benchmark,
  records swarm dashboard + final report. Should produce deterministic-ish
  output for replayable footage.
- **Animated CSV reveal:** SVG/AE animation that builds the headline numbers
  one row at a time.
- **Architecture animation:** simple block diagram showing how the LLM-based
  scorer compares actual findings vs expected findings.
- **Theme:** `apex` dark.
- **Resolution:** 1920×1080, 60fps. Terminal at left 60%, data-vis right 40%
  during split-screen scenes.

---

## Things to check first

- [ ] All three picked benchmarks reach `success` status (recall ≥ 1) on a dry
      run the day of recording. If a benchmark flakes, swap it for another.
- [ ] Docker Compose stacks for the three benchmarks reset cleanly between runs.
- [ ] The scoring rubric we narrate (recall, precision, status) matches what's
      in `benchmarks/argus/README.md` — read it once before recording.
- [ ] CSV reveal animation pulls live numbers from `consolidated-results.csv`,
      not hardcoded values. If we re-run benchmarks the video stays current.
- [ ] Cost figure ($1,190) and date range (Feb 2–13, 2026) are accurate; if
      newer runs exist, update narration.
- [ ] Final report PDF/MD for at least one benchmark is rendered in the report
      viewer to confirm it looks clean on camera.
- [ ] Confirm the `Run Date` column on the CSV is sorted/filtered to recent runs
      so the on-screen footage and the data file are consistent.

---

## Full script

### Scene 1 — Cold open (0:00–0:15)

**Visual:** Black frame. White text:

```
60 vulnerable apps.
One terminal.
```

Beat. Cut to a fast montage: 3 second-long clips of swarm dashboards crunching,
green checks ticking, finding cards popping.

**VO:** "Sixty vulnerable applications. One terminal. We ran the Argus
benchmark suite end-to-end against Apex. Here's what came back."

---

### Scene 2 — The headline numbers (0:15–0:55)

**Visual:** Animated CSV reveal. Header row appears. Then row by row, the
totals build:

```
Total benchmarks                          60
Completed with scored results             55
100% recall (all expected vulns found)    34
Failed to run (infra issues)               4
Total spend (Feb 2–13, 2026)         ~$1,190
```

**VO:** "Out of 60 benchmarks, 55 ran cleanly. Of those, Apex hit 100% recall —
caught every expected vulnerability — on 34 of them. Total cost: about twelve
hundred dollars across all runs."

Pull-quote callout from the report:

> "100% recall means the agent found every vulnerability the benchmark author
> planted. No misses."

---

### Scene 3 — Methodology, briefly (0:55–1:30)

**Visual:** Architecture animation. Two boxes — "Apex Findings" and "Expected
Findings" — feeding into a third box labeled "LLM Scorer." Output: matched /
unmatched.

**VO:** "Scoring is automated. Each benchmark ships with a list of expected
findings. After Apex runs, an LLM-based scorer matches what Apex reported
against what was expected — semantically, not by string compare. So a finding
called 'JWT signature bypass' matches an expected 'JWT confusion via alg=none'
when they describe the same bug at the same location."

Quick callout overlay:

```
Recall    = matched / expected
Precision = matched / actual reported
```

---

### Scene 4 — Live run #1: SQLi (1:30–2:15)

**Visual:** Split-screen. Left: terminal running
`pensar pentest --target http://argus-013.local`. Right: a small "expected
findings" panel from the benchmark spec showing one entry, "SQL injection in
`/products?id=`."

Time-lapse the run at 4×. Swarm spins up. A finding appears.

**VO:** "First benchmark: SQL injection. Expected findings: one. Apex finds it
in under three minutes — and the evidence shows the exact payload that worked."

When the finding appears, freeze frame and overlay the matched check:

```
✓ Expected: SQLi in /products?id=
✓ Reported: SQL injection on /products?id= (payload: 1' OR 1=1 --)
Recall: 100%
```

---

### Scene 5 — Live run #2: SSRF (2:15–3:00)

Same structure as Scene 4, different vulnerability. Pick a benchmark where
the swarm fans out to multiple sub-agents — visually richer.

**VO:** "Second benchmark: server-side request forgery. The agent fingerprints
the input, hypothesizes a metadata-service target, and confirms it. This is
the kind of test that takes a junior pentester an afternoon. Apex does it
while you're getting coffee."

---

### Scene 6 — Live run #3: a chain (3:00–3:45)

Pick a multi-step benchmark from the 041–060 range — the "chain" benchmarks
that require sequential exploitation.

**VO:** "Third benchmark — a chain. No single vulnerability gives you the
flag. You have to sequence them. This is where most automated scanners fall
over."

Time-lapse the run. Highlight the swarm dashboard's sub-agent fanout. Finish
on the chained finding card showing the full path: A → B → C.

---

### Scene 7 — Cost & honesty (3:45–4:20)

**Visual:** Cost-breakdown card pulled from `cost-breakdown.md`. Per-benchmark
cost histogram.

**VO:** "Honesty matters. Apex didn't get 100% recall on every benchmark — 21
of 55 came in below. The full report and the consolidated CSV are in the repo.
You can re-run any benchmark yourself."

Show the path on screen:

```
benchmarks/argus/consolidated-results.csv
benchmarks/argus/consolidated-report.md
github.com/pensarai/argus-validation-benchmarks
```

---

### Scene 8 — Outro (4:20–4:35)

**Visual:** Reuse the cold-open card, now with results overlaid:

```
60 vulnerable apps.
34 perfect runs.
One terminal.
```

**VO:** "Sixty apps. Thirty-four perfect runs. One install line. Try it
yourself — link in the description."

End card.

---

## Editing notes

- Slow zooms on data — let viewers absorb numbers.
- Always show the file path on screen when citing a public artifact (CSV,
  report) so people can verify.
- Avoid bragging about the 34. Pair every "we did this" with "and here's the
  number we missed" — it builds more trust than a clean win does.
- Keep all benchmark hostnames `.local` or `argus-NNN.local`. Never show real
  IPs.
- If we re-run benchmarks after publishing, re-render the CSV reveal and
  update the title overlay. Don't let public numbers drift.
