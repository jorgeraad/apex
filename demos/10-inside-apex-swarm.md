# Demo 10 — Inside Apex #1: why a swarm beats a mega-agent

**Length:** 7 minutes
**Format:** Whiteboard-style animated explainer w/ live footage cutaways
**Audience:** Sec-eng, technical decision-makers, agent-architecture enthusiasts
**Series:** **Inside Apex** — episode 1

---

## Purpose

Make the architecture argument for Apex's three-tier specialized-agent
swarm — the choice we made instead of "one big agent that does everything."
This episode is rooted in
[`decisions/PDR-003-specialized-agents.md`](../decisions/PDR-003-specialized-agents.md).

The viewer should walk away thinking: *"That's the right call, and they
thought hard about it."* This builds technical credibility with the
hardest-to-impress audience: people who build agents for a living.

---

## Requirements

- **Architecture animation** (After Effects or hand-drawn):
  - "One big agent" model — single box with everything piled in.
  - Three-tier hierarchy — `OffensiveSecurityAgent` at top, specialized
    agents in the middle (`TargetedPentestAgent`, `AttackSurfaceAgent`,
    `AuthenticationAgent`, `CodeAgent`), orchestration tools as connectors.
  - Animated swarm — multiple specialized agents fanning out to multiple
    targets in parallel.
- **Live swarm dashboard recording:** a real `/pentest` run with 5+
  sub-agents on a multi-target attack surface, captured at full quality.
- **Tokens-over-time chart:** an overlay that compares context-window growth
  in two scenarios:
  - One agent doing all 5 targets sequentially.
  - 5 specialized agents in parallel.
  Pull data from a real run if possible; otherwise illustrate honestly.
- **Findings registry stream:** a recording of the registry receiving writes
  from multiple agents simultaneously, rendered as a tickertape.
- **Theme:** `apex` dark for live footage; clean light theme for the
  explainer animation.

---

## Things to check first

- [ ] PDR-003 has been re-read recently and any narration aligns with the
      written rationale. We're representing the decision, not freelancing
      it.
- [ ] The architecture animation maps to *real* component names from the
      codebase: `OffensiveSecurityAgent`, `TargetedPentestAgent`,
      `AttackSurfaceAgent`, `AuthenticationAgent`, `CodeAgent`,
      `spawn_pentest_swarm`, `run_attack_surface`, `spawn_coding_agent`.
      Don't invent.
- [ ] The tokens-over-time chart is honest. If we can't measure both modes
      side by side, label the chart "illustrative."
- [ ] The recorded swarm run has at least 5 sub-agents and at least 3
      different specialized agent types active at once. Otherwise the visual
      argument is weaker than the verbal one.
- [ ] Findings registry tickertape captures real timestamped writes from
      multiple agents. Slow-mo helps the viewer see they're interleaved.
- [ ] No live footage shows real customer hostnames; all recordings target
      `vuln-shop` or Argus benchmarks.

---

## Full script

### Scene 1 — Cold open (0:00–0:25)

**Visual:** Whiteboard background. A single big circle is drawn, labeled
"one agent." Around it, tools fly in: recon, auth, exploit, code analysis,
report writer. The circle gets crowded.

**VO:** "When we started building Apex, the obvious approach was: one big
agent. Give it every tool, give it a long prompt, let it figure out the
engagement. We tried that. It worked badly. Today's video is about why,
and what we did instead."

---

### Scene 2 — The mega-agent failure mode (0:25–1:30)

**Visual:** Animated tokens-over-time chart. X-axis: engagement progress.
Y-axis: context window utilization. The line climbs steadily, then
flattens at the model's limit. Above the chart: snippets of the agent's
output, getting visibly less coherent.

**VO:** "First problem: context window fidelity. A pentest is long. By the
time the agent has crawled the surface, tested twenty endpoints, captured
fifty responses, and tried to chain a few exploits, its context is full
of recon noise — failed attempts, uninteresting responses, intermediate
artifacts. The model's recall on what actually matters drops. Late in the
engagement, exactly when reasoning matters most, the agent is at its
worst."

Pull-quote callout:

> "A specialized agent starts every run with a clean, focused context."
> — PDR-003

---

### Scene 3 — Specialization quality (1:30–2:30)

**Visual:** Whiteboard. Two prompt boxes side-by-side.

Left, labeled "general-purpose":

```
You are a pentester. Your tools are <40 tools>.
Your job is to test <broad scope>. Use <any
methodology you find useful>. Be thorough.
```

Right, labeled "AuthenticationAgent":

```
You are an authentication-flow specialist.
Your tools are <12 auth-relevant tools>.
You think only about: SSO, OAuth flows, session
fixation, MFA bypass, JWT confusion, race
conditions in auth state. You are not concerned
with anything else.
```

**VO:** "Second problem: specialization. The right authentication-flow
prompt is sharp. It encodes years of expertise about SSO, OAuth, session
fixation, MFA, JWT issues. You can't write that prompt and *also* the
prompt for SQL injection testing in the same agent without diluting both."

---

### Scene 4 — The three tiers (2:30–3:45)

**Visual:** Animation builds the hierarchy:

```
                  ┌─────────────────────────────┐
   Tier 1         │  OffensiveSecurityAgent     │ ← base harness
                  │  · tools                    │
                  │  · approval gate            │
                  │  · message persistence      │
                  │  · subagent callbacks       │
                  └────────┬───────────┬────────┘
                           │           │
                  ┌────────▼─┐    ┌────▼──────────┐
   Tier 2         │ Targeted │    │ Authentication │ ← specialized
                  │ Pentest  │    │ Agent          │   agents
                  │ Agent    │    └────────────────┘
                  └──────────┘
                           │
                  ┌────────▼─────────┐
   Tier 3         │ spawn_pentest_   │ ← orchestration
                  │ swarm tool       │   tools
                  └──────────────────┘
```

**VO:** "Three tiers. The base — `OffensiveSecurityAgent` — owns the
plumbing: tools, approval gate, message persistence, subagent callbacks.
Above that, specialized agents — `TargetedPentestAgent`,
`AuthenticationAgent`, `AttackSurfaceAgent`, `CodeAgent`. Each one is
expert at exactly one thing. And the connective tissue: orchestration
tools like `spawn_pentest_swarm` that the base agent can call to fan out
specialized agents at runtime, based on what it discovers."

---

### Scene 5 — Live swarm dashboard cutaway (3:45–4:45)

Cut from animation to recorded TUI swarm dashboard. 5 sub-agents active.
Sub-agent panes scroll independently. The base agent's status bar shows
which specialized agent is doing what.

**VO:** "Here's what that looks like running. Five specialized agents,
five different missions, all in parallel. The orchestrator decides at
runtime how many to spawn and what to give each one. You don't see this
shape in a hardcoded pipeline — it adapts based on what the discovery
phase finds."

---

### Scene 6 — The findings registry (4:45–5:30)

Cut to the findings registry tickertape — a fast scroll of writes coming
in from different agents, color-coded by source agent. Slow-mo so viewers
can see the interleaving.

**VO:** "All of these agents write to one shared findings registry in
real time. Deduplication happens at write — if two agents discover the
same SQLi from different angles, the registry merges them and keeps the
strongest evidence. The TUI subscribes to the registry and renders
findings as they land. The report at the end reads from one place. No
post-hoc merge. No conflicts. That's PDR-005, by the way."

---

### Scene 7 — When NOT to swarm (5:30–6:15)

**Visual:** Cut to `/operator` mode — single agent, chat-shaped. Quiet.

**VO:** "The swarm is for `/pentest` — wide coverage, autonomous. It's
not the right shape for `/operator`. Operator mode is one agent, every
tool, human at the wheel. You wouldn't fan out for an exploit chain — you
want all the context in one place so the operator can reason with it.
Different shape, different problem."

---

### Scene 8 — Outro (6:15–6:50)

**Visual:** Card.

```
INSIDE APEX · EPISODE 1
NEXT: THE 7-STEP STATE MACHINE
```

Path overlay:

```
decisions/PDR-003-specialized-agents.md
```

**VO:** "That's Inside Apex, episode one. The decision record is in the
repo — link in the description. Episode two: why we make the agent walk
through a strict seven-step state machine on every target."

---

## Editing notes

- This is the most "talking-architecture" video in the series. Lean on the
  animation to keep it visually alive.
- Every claim should be backed by a citation to PDR-003 or to a code path
  on screen. This episode's credibility depends on rigor.
- The tokens-over-time chart is the most likely thing to attract criticism
  from technical viewers. Either measure honestly or label "illustrative"
  — don't fudge.
- Don't make this video sales-y. It's an engineering talk. The audience
  for this video is the audience that buys based on engineering talks.
