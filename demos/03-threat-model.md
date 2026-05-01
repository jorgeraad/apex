# Demo 03 — `/threat-model` from clone to attacker's playbook in 3 minutes

**Length:** 3 minutes
**Format:** Screencast w/ voiceover
**Audience:** Security engineers, pentesters
**Series:** Standalone

---

## Purpose

Spotlight `/threat-model` — the whitebox feature that almost no other tool in
the agentic-pentest space ships well. The viewer should walk away thinking:
*"My security team should be running this every quarter."*

Secondary goal: set up the workflow link to Demo 15 (whitebox + blackbox
combined). The threat model produced here is the input to a `--threat-model`
flag on a later pentest.

---

## Requirements

- **Asset:** "threat-model target" — a small open-source-style web app
  (Node/Express + React frontend recommended, ~3K LOC) with deliberate but
  realistic security-relevant architecture choices: JWT auth, file-upload
  endpoint, admin route, third-party SaaS integration. Should feel real, not
  contrived.
- **Anthropic API key** with enough credits for ~300K tokens of code analysis.
- **Output viewer:** the resulting `threat-model.md` should render cleanly in
  the report viewer dialog *and* in a markdown viewer (VS Code) — we'll cut
  between both.
- **Animated phase indicator:** an overlay that shows which of the eight
  threat-modeling phases the agent is currently in.
- **Theme:** `apex` dark.

---

## Things to check first

- [ ] Run `/threat-model` against the target a day before recording — confirm
      output is high quality and the agent doesn't flake on any phase.
- [ ] The target has *something* in every phase. If it has no admin route,
      Phase 5 (privileged surfaces) will be thin and the demo loses momentum.
- [ ] Codebase path is short and shows well on screen
      (`~/demo/threat-model-target/`, not a 14-segment path).
- [ ] Disable any IDE indexers or shell prompts that pollute the terminal.
- [ ] Confirm the resulting threat model includes attack-path tables — those
      are the visual hero shot.
- [ ] The eight-phase animation matches the actual phases the prompt walks
      through (read `src/core/skills/builtins/threatModel.ts` for the source
      of truth).
- [ ] The output file path used in the demo (`./tm.md`) is actually written —
      `create_file` with `overwrite: true` lands at the working directory.

---

## Full script

### Scene 1 — Cold open (0:00–0:12)

**Visual:** Black frame. White text appears one line at a time:

```
Before you pentest...
you should know what you're testing.
```

Cut to a terminal with a freshly cloned repo. `ls` shows a typical web app:
`src/`, `package.json`, `README.md`.

**VO:** "Before you pentest a thing, you should know what you're testing.
That's the threat model. Apex generates one for you."

---

### Scene 2 — Setup (0:12–0:30)

**Type:**

```bash
cd ~/demo/threat-model-target
pensar
```

TUI boots, settles in operator view. Type the slash command:

```
/threat-model --output ./tm.md
```

**VO:** "Run `/threat-model` from the codebase root. Output path is whatever
you want — we'll write it as `tm.md` here."

---

### Scene 3 — The eight phases (0:30–2:15)

**Visual:** Operator view streams. Phase indicator overlay top-right of the
screen, ticking through:

```
Phase 1 — Application identity
Phase 2 — Trust boundaries
Phase 3 — Data flows
Phase 4 — Auth & sessions
Phase 5 — Privileged surfaces
Phase 6 — External dependencies
Phase 7 — Secrets & key material
Phase 8 — Attack paths
```

For each phase, show 2–4 seconds of the agent reading actual files:
`package.json`, `src/auth.ts`, `src/upload.ts`, etc. Don't fake this — the
agent really does open files. Let viewers see file paths flicker.

**VO (over the phases, paced):** "Apex doesn't guess. It reads every file that
matters: configs, route definitions, auth middleware, dependency lists. Phase
by phase, it builds a picture of what the app *is* before it asks how the app
might *break*. Application identity. Trust boundaries. Data flows. Auth and
sessions. Privileged surfaces. External dependencies. Secrets. And then the
final phase — attack paths."

When Phase 8 starts, slow the playback. The agent is now reasoning across
phases.

---

### Scene 4 — The output (2:15–2:45)

Agent finishes. Last message:

```
Threat model written to ./tm.md (12,400 words, 7 attack paths)
```

Cut to VS Code's markdown preview of `tm.md`. Scroll through:
- Application identity table.
- Trust-boundary diagram (Mermaid, rendered).
- Auth flow walkthrough.
- The hero shot: an "Attack Paths" table with columns like *Entry point*,
  *Privilege required*, *Steps*, *Impact*.

**VO:** "Twelve thousand words. Seven attack paths. Each one ties an entry
point to an impact, with the steps in between. This is what a senior
application security architect would write — except they'd take a week."

---

### Scene 5 — The handoff (2:45–3:00)

Cut back to terminal. Type:

```bash
pensar pentest --target https://staging.threat-model-target.local \
  --threat-model @./tm.md
```

Don't run it — just type and pause.

**VO:** "And here's the kicker: this file becomes the input to your next
pentest. `--threat-model` tells Apex what to focus on, so the agent doesn't
waste time on things you already know are safe."

End card:

```
/threat-model
ship/quarterly/before-every-major-release
```

---

## Editing notes

- The phase indicator is critical — it gives structure to what would
  otherwise be a wall of streaming text.
- When showing the agent reading files, briefly highlight the file path with
  a subtle yellow underline so viewers track what's happening.
- The Mermaid diagram in the output is the moment that wows non-technical
  viewers. Pause on it for two beats.
- Don't read the attack-paths table aloud — let viewers pause if they want
  to. Just narrate the meta.
- Tease Demo 15 (whitebox + blackbox combo) in the description, not the
  video itself.
